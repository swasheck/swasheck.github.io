---
author: swasheck
comments: true
date: 2014-11-12T04:07:55+00:00
layout: post
link: https://swasheck.wordpress.com/2014/11/11/sql-server-and-refs-part-1-dbcc-and-in-memory-oltp/
slug: sql-server-and-refs-part-1-dbcc-and-in-memory-oltp
title: 'SQL Server and ReFS: Part 1 - DBCC and In Memory OLTP'
wordpress_id: 273
tags:
- ReFS
- SQL Server 2014
categories: 
- "SQL Server"
---

As I was sitting in Bob Ward's Inside SQL Server I/O presentation, something interesting caught my eye on a particular slide.

[![ward](https://swasheck.files.wordpress.com/2014/11/ward.png?w=300)](http://msdn.microsoft.com/en-us/library/windows/desktop/hh965610(v=vs.85).aspx)

It looks like ReFS is now supported for SQL Server 2014. I'd run into problems with 2012 so I'd just given up but this looks promising. I am neither a filesystem aficionado, nor a dilettante but I know that there are some interesting features of ReFS at which Windows server admins are looking to see if it's viable. 

There are a few known gotchas with ReFS that I'm going to (hopefully) test, along with performance characteristics. Performance will be addressed in a separate post because I've already got the numbers and the post will be a bit deeper and less along the lines of "work/no-work."
<!-- more -->
Back in July, 2014,[ Qumio gave In-Memory OLTP a try](http://www.qumio.com/Blog/Lists/Posts/Post.aspx?ID=14) with the ReFS filesystem without success. It's been a few months, SQL Server 2014 is out of CTP, and I trust Bob Ward (and his caveat to disable integrity checks as they can cause unintended data file corruption). Additionally, the [ReFS documentation](http://technet.microsoft.com/en-us/library/hh831724.aspx) notes that alternate data stream support has been added to ReFS, so let's test.

Quick-format a volume (j:, in this case) with ReFS, disabling Integrity Steam checks.
[![fmt](https://swasheck.files.wordpress.com/2014/11/fmt.png?w=300)](https://swasheck.files.wordpress.com/2014/11/fmt.png)

For fun, I wanted to see would happen with different file layout configurations, with at least one being on an ReFS-formatted volume (j:\ and p:\ are my ReFS volumes, the others are NTFS):

[code language="sql"]
-- note, the SQL Server Service account has the Perform Volume Maintenance Tasks privilege on the OS
create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'j:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'l:\logs\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);

-- SQL Server Execution Times:
--   CPU time = 15 ms,  elapsed time = 5675 ms.
   
 create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'j:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'j:\data\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);
 
 -- SQL Server Execution Times:
 --   CPU time = 30 ms,  elapsed time = 10522 ms.   
   
  create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'm:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'j:\data\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);

 -- SQL Server Execution Times:
 --   CPU time = 31 ms,  elapsed time = 2566 ms

    create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'm:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'l:\logs\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);

 -- SQL Server Execution Times:
 --   CPU time = 47 ms,  elapsed time = 2737 ms.

create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'j:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'p:\logs\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);
-- SQL Server Execution Times:
--   CPU time = 16 ms,  elapsed time = 6773 ms.

create database refs_test 
	on (
		name = refs_test_dat, 
		filename = 'm:\data\refs_test.mdf',
		size = 25GB,
		maxsize = 45GB,
		filegrowth = 1GB
	)
	log on (
		name = refs_test_log,
		filename = 'm:\data\refs_test.ldf',
		size = 1GB,
		maxsize = 10GB,
		filegrowth = 1GB
	);
-- SQL Server Execution Times:
--   CPU time = 47 ms,  elapsed time = 20573 ms.
[/code]

(Each DDL was executed five times and the best run of each was taken. No averages, variances, or standard deviations were harmed in this exercise.)

So the fastest database creation occurred with the log on ReFS and the data file on NTFS. The slowest was with all files on the same volume with ReFS edging NTFS in that regard. This looks like an area for exploration with Bob's PASS Summit 2014 WinDbg scripts.

Now let's see if we can do some In-Memory OLTP.

[code language="sql"]
alter database refs_test
	add filegroup refs_test_memopt contains memory_optimized_data;

alter database refs_test
	add file (
		name = refs_test_memopt_file,
		filename = 'j:\data\refs_test_memopt_file'
	) to filegroup refs_test_memopt;

alter database refs_test
	set memory_optimized_elevate_to_snapshot=on;

[/code]
[![addfg](https://swasheck.files.wordpress.com/2014/11/addfg1.png?w=300)](https://swasheck.files.wordpress.com/2014/11/addfg.png)
No errors.

Now let's create a memory-optimized table and populate it with some data:
[code language="sql"]
use refs_test;
go

CREATE TABLE dbo.lemma(
	lemma_id int identity(1,1) not null primary key nonclustered hash with (bucket_count = 400000),
	lemma nvarchar(255) null
) with (memory_optimized=on);

-- can't do a cross-database transaction with a memory-optimized table
select *
into dbo.l2
from greek.dbo.lemma;

insert lemma (lemma)
select lemma from l2

drop table l2;

select *
into dbo.f_word -- create a non-memory optimized table
from greek.dbo.f_word;
[/code]
[![popdata](https://swasheck.files.wordpress.com/2014/11/popdata.png?w=300)](https://swasheck.files.wordpress.com/2014/11/popdata.png)

That works as well. What about creating a database snapshot?

[code language="sql"]
use master;
go

create database greek_snap on (
	name = greek_Data, filename='j:\data\greek_snap.ss'
) as snapshot of greek;
	
select 
	d.name,
	mf.name,
	mf.physical_name,
	type_desc,
	data_space_id, 
	is_sparse, 
	is_memory_optimized_elevate_to_snapshot_on
from sys.master_files mf
join sys.databases d 
	on mf.database_id = d.database_id
where d.database_id > 4;
[/code]
[![snap](https://swasheck.files.wordpress.com/2014/11/snap.png?w=300)](https://swasheck.files.wordpress.com/2014/11/snap.png)
Well that worked well and we can see a sparse file sitting on 

Still no problems. What about a DBCC CHECKDB?

[code language="sql"]
use master;
dbcc checkdb(refs_test);
[/code]
We can see that the Memory-Optimized table is skipped, but the other table is not skipped and is checked, but it is done without any errors.
[![dbcc](https://swasheck.files.wordpress.com/2014/11/dbcc.png?w=300)](https://swasheck.files.wordpress.com/2014/11/dbcc.png) 

[Recalling a previous blog post](http://swasheck.wordpress.com/2013/10/07/more-information-on-dbcc-checks-with-new-extended-events-in-sql-server-2014-ctp1/) (and cannibalizing its code), let's check to see how CHECKDB is performed. Does it create a snapshot or is it taking out TABLOCKX locks while running?

[code language="sql"]
-- create the event session
create event session [DBCC_Check] on server
    add event sqlserver.check_phase_tracing(
        action(sqlserver.database_id,
            sqlserver.database_name)),
    add event sqlserver.check_thread_message_statistics(
        action(sqlserver.database_id,
            sqlserver.database_name)),
    add event sqlserver.check_thread_page_io_statistics(
        action(sqlserver.database_id,
            sqlserver.database_name)),
    add event sqlserver.check_thread_page_latch_statistics(
        action(sqlserver.database_id,
            sqlserver.database_name))
    add target package0.ring_buffer
        WITH (track_causality = on);


-- start the event session
alter event session [DBCC_Check] on server
	state = start;

-- run the dbcc
dbcc checkdb(refs_test);

-- parse the data in the ring buffer
select *    
    from (
            select
                event_data.value('(event/@name)[1]', 'varchar(50)') AS event_name,
                DATEADD(hh,DATEDIFF(hh, GETUTCDATE(), CURRENT_TIMESTAMP),
                event_data.value('(event/@timestamp)[1]', 'datetime2')) AS [timestamp],
                event_data.value('(event/data[@name="database_id"]/value)[1]', 'sysname') AS event_database_id,
                event_data.value('(event/action[@name="database_id"]/value)[1]', 'sysname') AS action_database_id,
                event_data.value('(event/data[@name="opcode"]/text)[1]', 'NVARCHAR(25)') AS opcode_text,
                event_data.value('(event/data[@name="call_duration"]/value)[1]', 'bigint') AS call_duration,
                event_data.value('(event/data[@name="is_remote"]/value)[1]', 'bit') AS is_remote,
                event_data.value('(event/data[@name="command_phase"]/text)[1]', 'nvarchar(100)') AS command_phase_text,
                event_data.value('(event/data[@name="logical_reads"]/value)[1]', 'bigint') AS logical_reads,
                event_data.value('(event/data[@name="physical_reads"]/value)[1]', 'bigint') AS physical_reads,
                event_data.value('(event/data[@name="run_ahead_reads"]/value)[1]', 'bigint') AS run_ahead_reads,
                event_data.value('(event/data[@name="total_page_io_latch_waits"]/value)[1]', 'bigint') AS total_page_io_latch_waits,
                event_data.value('(event/data[@name="page_io_latch_wait_time_in_ms"]/value)[1]', 'bigint') AS page_io_latch_wait_time_in_ms,
                event_data.value('(event/data[@name="total_page_latch_waits"]/value)[1]', 'bigint') AS total_page_latch_waits,
                event_data.value('(event/data[@name="page_latch_wait_time_in_ms"]/value)[1]', 'bigint') AS page_latch_wait_time_in_ms,
                event_data.value('(event/data[@name="messages_sent"]/value)[1]', 'int') AS messages_sent,
                event_data.value('(event/data[@name="messages_received"]/value)[1]', 'int') AS messages_received,
                event_data.value('(event/action[@name="database_name"]/value)[1]', 'sysname') AS database_name,
                CAST(SUBSTRING(event_data.value('(event/action[@name="attach_activity_id"]/value)[1]', 'varchar(50)'), 1, 36) AS uniqueidentifier) as activity_id,
                CAST(SUBSTRING(event_data.value('(event/action[@name="attach_activity_id"]/value)[1]', 'varchar(50)'), 38, 10) AS int) as event_sequence,
                CAST(SUBSTRING(event_data.value('(event/action[@name="attach_activity_id_xfer"]/value)[1]', 'varchar(50)'), 1, 36) AS uniqueidentifier) as activity_id_xfer
            from
                ( select XEvent.query('.') AS event_data
                    from
                        ( -- Cast the target_data to XML
                            select CAST(target_data AS XML) AS TargetData
                                from sys.dm_xe_session_targets st
                                    JOIN sys.dm_xe_sessions s
                                        ON s.address = st.event_session_address
                                where name = 'DBCC_Check'
                                    AND target_name = 'ring_buffer'
                        ) AS Data
                        -- Split out the Event Nodes
                        CROSS APPLY TargetData.nodes ('RingBufferTarget/event') AS XEventData (XEvent)
                ) AS tab (event_data)
        ) xedata
        order by timestamp

-- stop the event session
alter event session [DBCC_Check] on server
	state = stop;

-- drop the event session
drop event session [DBCC_Check] on server;
[/code]

[![dbcc2](https://swasheck.files.wordpress.com/2014/11/dbcc2.png)](https://swasheck.files.wordpress.com/2014/11/dbcc2.png)

The row outlined in green shows the start of the step a snapshot is being created (so it actually creates a snapshot). The "action" database_id, outlined in red, is the database where the DBCC check is taking place - the snapshot. 

Based on this evidence, it looks like ReFS is supported by SQL Server 2014 for at least basic utilization. There may be things that are yet undiscovered in what amounts to a v1.0 release of a new filesystem, so I wouldn't necessarily run out and put all of my production SQL Servers on 2014 + ReFS. However, it's looking like support is certainly advancing.

Next time I'll talk IOPS and throughput using fio.
