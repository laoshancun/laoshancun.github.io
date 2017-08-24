---
author: laoshancun
comments: true
date: 2017-08-24 09:19:42+08:00
layout: post
slug: "sqlserver-optimization"
title: "sqlserver常用优化语句"
wordpress_id: 2
description: "sqlserver常用优化，性能监测语句"
categories:
- database
tags:
- sqlserver
- 优化
- 数据库
---


### 1. 查询某个数据库的连接数 ###
```sql 
select count(*) from Master.dbo.SysProcesses where dbid=db_id();
``` 

### 2. 前10名其他等待类型 ###
```sql 
SELECT TOP 10 * from sys.dm_os_wait_stats
ORDER BY wait_time_ms DESC;

SELECT * FROM sys.dm_os_wait_stats WHERE wait_type like 'PAGELATCH%'
OR wait_type like 'LAZYWRITER_SLEEP%';
```
### 3. CPU的压力 ###

```sql
SELECT scheduler_id, current_tasks_count, runnable_tasks_count
FROM sys.dm_os_schedulers
WHERE scheduler_id < 255;

--表现最差的前10名使用查询
SELECT TOP 10 ProcedureName = t.text,
ExecutionCount = s.execution_count,
AvgExecutionTime = isnull ( s.total_elapsed_time / s.execution_count, 0 ),
AvgWorkerTime = s.total_worker_time / s.execution_count,
TotalWorkerTime = s.total_worker_time,
MaxLogicalReads = s.max_logical_reads,
MaxPhysicalReads = s.max_physical_reads,
MaxLogicalWrites = s.max_logical_writes,
CreationDateTime = s.creation_time,
CallsPerSecond = isnull ( s.execution_count / datediff ( second , s.creation_time, getdate ()), 0 )
FROM sys.dm_exec_query_stats s
CROSS APPLY sys.dm_exec_sql_text( s.sql_handle ) t ORDER BY
s.max_physical_reads DESC;

SELECT SUM(signal_wait_time_ms) AS total_signal_wait_time_ms /*总信号等待时间 */,
SUM(wait_time_ms - signal_wait_time_ms) AS resource_wait_time_ms/*资源的等待时间*/,
SUM(signal_wait_time_ms) * 1.0 / SUM (wait_time_ms) * 100 AS [signal_wait_percent/*信号等待%*/],
SUM(wait_time_ms - signal_wait_time_ms) * 1.0 / SUM (wait_time_ms) * 100 AS [resource_wait_percent/*资源等待%*/]
FROM sys.dm_os_wait_stats;
```
一个信号等待时间过多对资源的等待时间那么你的CPU是目前的一个瓶颈。

### 4. 查看进程所执行的SQL语句 ###
```sql
if (select COUNT(*) from master.dbo.sysprocesses) > 500
begin
select text,CROSS APPLY master.sys.dm_exec_sql_text(a.sql_handle) from master.sys.sysprocesses a;

end
select text,a.* from master.sys.sysprocesses a
CROSS APPLY master.sys.dm_exec_sql_text(a.sql_handle)
where a.spid = '51'
dbcc inputbuffer(53)
with tb
as
(
select blocking_session_id,
session_id,db_name(database_id) as dbname,text from master.sys.dm_exec_requests a
CROSS APPLY master.sys.dm_exec_sql_text(a.sql_handle)
),
tb1 as
(
select a.,login_time,program_name,client_interface_name,login_name,cpu_time,memory_usage8 as 'memory_usage(KB)',
total_scheduled_time,reads,writes,logical_reads
from tb a inner join master.sys.dm_exec_sessions b
on a.session_id=b.session_id
)
select a.*,connect_time,client_tcp_port,client_net_address from tb1 a inner join master.sys.dm_exec_connections b on a.session_id=b.session_id
```

### 5. 当前进程数 ###
```sql
select * from master.dbo.sysprocesses
order by cpu desc
```
### 6. 查看当前活动的进程数 ###
```sql
sp_who active
```
### 7. 查询是否由于连接没有释放引起CPU过高 ###
```sql
select * from master.dbo.sysprocesses
where spid> 50
and waittype = 0x0000
and waittime = 0
and status = 'sleeping '
and last_batch < dateadd(minute, -10, getdate())
and login_time < dateadd(minute, -10, getdate())
```
### 8. 强行释放空连接 ###
```sql
select 'kill ' + rtrim(spid) from master.dbo.sysprocesses
where spid> 50
and waittype = 0x0000
and waittime = 0
and status = 'sleeping '
and last_batch < dateadd(minute, -60, getdate())
and login_time < dateadd(minute, -60, getdate())
```

### 9. 查看当前占用 cpu 资源最高的会话和其中执行的语句(及时CPU) ###
```sql
select spid,cmd,cpu,physical_io,memusage,
(select top 1 [text] from ::fn_get_sql(sql_handle)) sql_text
from master..sysprocesses order by cpu desc,physical_io desc
```
### 10. 查看缓存中重用次数少，占用内存大的查询语句（当前缓存中未释放的）--全局 ###
```sql
SELECT TOP 100 usecounts, objtype, p.size_in_bytes,[sql].[text]
FROM sys.dm_exec_cached_plans p OUTER APPLY sys.dm_exec_sql_text (p.plan_handle) sql
ORDER BY usecounts,p.size_in_bytes desc
SELECT top 25 qt.text,qs.plan_generation_num,qs.execution_count,dbid,objectid
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(sql_handle) as qt
WHERE plan_generation_num >1
ORDER BY qs.plan_generation_num
SELECT top 50 qt.text AS SQL_text ,SUM(qs.total_worker_time) AS total_cpu_time,
SUM(qs.execution_count) AS total_execution_count,
SUM(qs.total_worker_time)/SUM(qs.execution_count) AS avg_cpu_time,
COUNT(*) AS number_of_statements
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) as qt
GROUP BY qt.text
ORDER BY total_cpu_time DESC --统计总的CPU时间
--ORDER BY avg_cpu_time DESC --统计平均单次查询CPU时间
```
### 11. 计算可运行状态下的工作进程数量 ###
```sql
SELECT COUNT(*) as workers_waiting_for_cpu,s.scheduler_id
FROM sys.dm_os_workers AS o
INNER JOIN sys.dm_os_schedulers AS s
ON o.scheduler_address=s.scheduler_address
AND s.scheduler_id<255
WHERE o.state='RUNNABLE'
GROUP BY s.scheduler_id

SELECT creation_time N'语句编译时间'
,last_execution_time N'上次执行时间'
,total_physical_reads N'物理读取总次数'
,total_logical_reads/execution_count N'每次逻辑读次数'
,total_logical_reads N'逻辑读取总次数'
,total_logical_writes N'逻辑写入总次数'
, execution_count N'执行次数'
, total_worker_time/1000 N'所用的CPU总时间ms'
, total_elapsed_time/1000 N'总花费时间ms'
, (total_elapsed_time / execution_count)/1000 N'平均时间ms'
,SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
((CASE statement_end_offset
WHEN -1 THEN DATALENGTH(st.text)
ELSE qs.statement_end_offset END
- qs.statement_start_offset)/2) + 1) N'执行语句'
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
where SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
((CASE statement_end_offset
WHEN -1 THEN DATALENGTH(st.text)
ELSE qs.statement_end_offset END
- qs.statement_start_offset)/2) + 1) not like '%fetch%'
ORDER BY total_elapsed_time / execution_count DESC
```
### 12. 表空间大小查询 ###
```sql
create table #tb(N'表名' sysname,N'记录数' int,N'保留空间' varchar(100),N'使用空间' varchar(100),N'索引使用空间' varchar(100),N'未用空间' varchar(100))
insert into #tb exec sp_MSForEachTable 'EXEC sp_spaceused ''?'''
select * from #tb
go
SELECT
N'表名',
N'记录数',
cast(ltrim(rtrim(replace(N'保留空间','KB',''))) as int)/1024 N'保留空间MB',
cast(ltrim(rtrim(replace(N'使用空间','KB',''))) as int)/1024 N'使用空间MB',
cast(ltrim(rtrim(replace(N'使用空间','KB',''))) as int)/1024/1024.00 N'使用空间GB',
cast(ltrim(rtrim(replace(N'索引使用空间','KB',''))) as int)/1024 N'索引使用空间MB',
cast(ltrim(rtrim(replace(N'未用空间','KB',''))) as int)/1024 N'未用空间MB'
FROM #tb
WHERE cast(ltrim(rtrim(replace(N'使用空间','KB',''))) as int)/1024 > 0
--order by N'记录数' desc
ORDER BY N'使用空间MB' DESC
DROP TABLE #tb
```