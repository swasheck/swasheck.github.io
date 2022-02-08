---
author: swasheck
date: "2016-11-28T18:35:52+00:00"
title: 'Index Maintenance Operations'
categories: 
- "SQL Server"
tags:
- "Index Maintenance"
- "Extended Events"
- "Adventures in Index Maintenance Land"
---

Recently, our index maintenance process was called into question on an Availability Group because it began running significantly longer than expected. The initial theory put forward by other groups was that this was due the Availability Group which is a 2+1 cluster (2 HA synchronous replicas, 1 asynchronous DR replica) and there was a commit delay. Our monitoring tool [SentryONE](https://sentryone.com) didn't alert us to any significant delays, but we committed to investigate the theory since it would be a good exercise for all involved. However, our priority was to actually investigate the objects and indexes on which maintenance was done. Because we log the process' details, we were able to identify that our process had progressively skewed toward reorganization and away from rebuild operations. Additionally, the larger of the largest of the indexes (by page count and row count) had become 100% reorganization operations. 

Since nobody on our team was particularly familiar with the internals of each operation, I did a bit of research and came across [Tim Chapman's 2013 Summit Index Internals Deep Dive presentation](http://www.sqlpass.org/summit/2013/Sessions/SessionDetails.aspx?sid=5158). In the slide deck I read a few things that provided some clues into what was occurring during our maintenance process. The first is that reorganization is always single-threaded (slide 29) and that reorganization "[g]enerates a LOT more t-log records than Rebuild." Given this information, it would seem to make sense as to why our process had suddenly begun taking so much longer.

 - 1. Our larger tables were only being reorganized and not being rebuilt. Since reorganization is single-threaded, moving through high-page count tables on one thread would naturally take a bit longer. 
 - 2. If what Tim said is true about t-log records, then there could potentially be a greater chance of log commit delay on the Availability Group, though it hadn't become problematic enough for us to experience a breach of our alert threshold. 

Certainly this is helpful information and served as a good initial response. However, we need to be able to come up with a better explanation, and we need to validate what we found on the Internet. To do this, I created a test scenario in which I'd test the two operations on an index and see how the server responds. The process is as follows:

 - 1. Backup the database
 - 2. Rebuild the candidate index
 - 3. Restore the database
 - 4. Reorganize the candidate index

To monitor these processes I created an Extended Event session with many of the conceivable events that could be raised during the processes (I collected additional actions and filtered based on the database in question). 

```
create event session XEIndexRebuild on server 
	add event sqlos.wait_info,
	add event sqlserver.auto_stats,
	add event sqlserver.databases_dbcc_logical_scan,
	add event sqlserver.databases_log_flush,
	add event sqlserver.degree_of_parallelism,
	add event sqlserver.file_write_completed,
	add event sqlserver.index_build_extents_allocation,
	add event sqlserver.lock_released(set collect_resource_description=(1)),
	add event sqlserver.locks_lock_waits,
	add event sqlserver.object_altered,
	add event sqlserver.oiblob_cleanup_end,
	add event sqlserver.physical_page_read,
	add event sqlserver.physical_page_write,
	add event sqlserver.scan_stopped,
	add event sqlserver.sql_batch_completed,
	add event sqlserver.sql_batch_starting,
	add event sqlserver.transaction_log
add target package0.event_file(set filename=N'XEIndexRebuild')
with (
            max_memory=4096 kb,
            event_retention_mode=allow_single_event_loss,
            max_dispatch_latency=30 seconds,
            max_event_size=0 kb,
            memory_partition_mode=none,
            track_causality=on,
            startup_state=off
    )
go
```

![tlog usage](https://swasheck.gitlab.io/fixed/2016-11-28_transaction_log.png)

The test subject index is a nonclustered index with 37,108 pages and 1,904,887 rows on a SQL Server 2016 instance. Rebuilding this index generated 3.83 MB over 55,826 log records, while Reorganization generated 215.86 MB over 3,750,552 log records. On the same index with the same level of fragmentation, Reorganization generates many more log records (67.18 times more entries and 56.36 more data) to ship to secondaries. However, since this initial test was not done on an Availability Group, it's not entirely clear what impact the additional data has on the remote harden process with the synchronous replica.

![build sequence](https://swasheck.gitlab.io/fixed/2016-11-28_reorg_reads_writes.png) 

A big reason for the log volume disparity is how each of these operations performs its task. Where Rebuild, "drops and re-creates the index", Reorganize "defragments the leaf level of clustered and nonclustered indexes on tables and views by physically reordering the leaf-level pages to match the logical, left to right, order of the leaf nodes" and compacts the pages [MSDN](https://msdn.microsoft.com/en-us/library/ms189858.aspx). Tracking the transaction log and page reads using our event session, we can see the engine start reading index pages and read sequentially through the pages. It periodically opens a system transaction (seen in the `transaction_log` event) to write out the pages in their l-t-r sequence, confirming the MSDN documentation.  

![distinct threads](https://swasheck.gitlab.io/fixed/2016-11-28_distinct_threads.png)

Finally, the other claim that we need to verify is that Reorganize is a single-threaded operation, while Rebuild can be parallel. It is possible to verify the claim that Reorganize is a single-threaded operation a few different ways. The first way I verified this was by counting the number of distinct system threads throughout the process. Additionally, I used the `degree_of_parallelism` event, but it was never fired for the Reorganize operation. The Rebuild operation did fire the `degree_of_parallelism` event with DOP of 4.

We've proven Tim Chapman's presentation to be true and accurate and have confirmed at least one bit of information from the MSDN documentation. However, there are many improvements to the experiment in question. My Adventures in Index Maintenance-Land have begun.