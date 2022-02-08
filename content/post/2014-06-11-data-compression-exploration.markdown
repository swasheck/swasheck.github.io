---
author: swasheck
comments: true
date: 2014-06-11T20:39:51+00:00
layout: post
link: https://swasheck.wordpress.com/2014/06/11/data-compression-exploration/
slug: data-compression-exploration
title: Data Compression Exploration
wordpress_id: 244
categories: 
- "SQL Server"
tags:
- Compression
---

I recently wanted to explore potential candidates for data compression in our environment. I found a few ways to go about doing this, most notably this [TechNet article](http://technet.microsoft.com/en-us/library/dd894051(v=SQL.100).aspx) that steps through strategy and planning. This was helpful, but I wanted to come up with a repeatable process. Since we have partitioned tables, I also wanted to examine candidates for compression at the partition level for each index.

<!-- more -->

After conversation with some people whom I consider smarter than I am, I decided that my magic number would be 3. That is, I wanted a 3:1 read:write ratio for a partition to even be considered for compression and only then would it be tested using [sp_estimate_data_compression_savings ](http://msdn.microsoft.com/en-us/library/cc280574.aspx)for at least a 3:1 compression ratio.

[code lang="sql"]
/*
	Clean up from any past runs
*/
if exists (select 1 from tempdb.sys.tables where name = 'compression_estimate')
	begin
		drop table  tempdb.dbo.compression_estimate;
	end
if exists (select 1 from tempdb.sys.tables where name = 'compression_candidates')
	begin
		drop table  tempdb.dbo.compression_candidates;
	end

-- Create the object that will hold the compression estimates

create table tempdb.dbo.compression_estimate (
	object_name sysname,
	schema_name sysname,
	index_id int,
	partition_number int,
	size_with_current_compression_settings decimal(23,2),
	size_with_requested_compression_setting  decimal(23,2),
	sample_size_with_current_compression  decimal(23,2),
	sample_size_with_requested_compression  decimal(23,2)
);

-- Run the base query to get the initial candidates (3:1 read:write ratio)

select * ,
	case writes
		when 0 then reads
		else (reads/(1.*writes))
	end as rw_ratio,
	'EXEC sp_estimate_data_compression_savings ''' + schema_name + ''',''' + table_name
		+ ''',' + cast(index_id as nvarchar(5)) + ',' + cast(partition_number as nvarchar(5)) +  ',''PAGE''' compression_sql
	into tempdb.dbo.compression_candidates
	from (
			select
				t.name table_name,
				i.name index_name,
				i.index_id,
				object_schema_name(i.object_id) schema_name,
				p.partition_number,
				rows,
				ios.leaf_insert_count + ios.leaf_delete_count + ios.leaf_update_count writes,
				ios.range_scan_count + ios.singleton_lookup_count reads,
				au.used_pages,
				au.total_pages,
				count(*) buffered_pages
				from sys.dm_os_buffer_descriptors bd
				join sys.allocation_units au
					on bd.allocation_unit_id = au.allocation_unit_id
				join sys.partitions p
					on au.container_id = p.hobt_id
					and  (au.type = 1 OR au.type = 3)
				join sys.indexes i
					on p.object_id = i.object_id
					and p.index_id = i.index_id
				join sys.tables t
					on i.object_id = t.object_id
				cross apply  sys.dm_db_index_operational_stats(DB_ID(),NULL,NULL,NULL) ios
			where bd.database_id = DB_ID()
				and ios.object_id = p.object_id
				and ios.index_id = p.index_id
				and ios.partition_number = p.partition_number
			group by t.name,
				i.name,
				i.object_id,
				i.index_id,
				p.partition_number,
				rows,
				ios.leaf_insert_count + ios.leaf_delete_count + ios.leaf_update_count ,
				ios.range_scan_count + ios.singleton_lookup_count ,
				au.used_pages,
				au.total_pages
	) base_agg
where case writes
		when 0 then reads
		else (reads/(1.*writes))
	end > 3
option (recompile, maxdop 1);

/*
	Declare the local variables and cursor to iterate the candidates to estimate their compression values
*/
declare @schema_name sysname, @table_name sysname, @index_id int, @partition_number int;

declare page_cur cursor fast_forward read_only for
	select schema_name,table_name,index_id,partition_number
	from tempdb.dbo.compression_candidates

open page_cur
fetch next from page_cur
	into @schema_name , @table_name , @index_id , @partition_number ; 

while @@fetch_status = 0
	begin
		insert into tempdb.dbo.compression_estimate
			exec sp_estimate_data_compression_savings @schema_name,@table_name,@index_id,@partition_number,'PAGE';
		fetch next from page_cur
			into @schema_name , @table_name , @index_id , @partition_number ;
	end
close page_cur
deallocate page_cur

/*
	Alter the base table to add the compression estimates
	(could just join on the candidates on the estimates but didn't)
*/
alter table tempdb.dbo.compression_candidates
	add current_size decimal(23,2), compressed_size decimal(23,2);

update c
set
	current_size = size_with_current_compression_settings,
	compressed_size = size_with_requested_compression_setting
from tempdb.dbo.compression_candidates c
	join tempdb.dbo.compression_estimate e
		on c.schema_name = e.schema_name
		and c.table_name = e.object_name
		and c.index_id = e.index_id
		and c.partition_number = e.partition_number

-- pull a report and generate the sql for compression implementation
select *,
	'alter index ' + quotename(index_name) + ' on ' + quotename(table_name) +
		' rebuild partition = ' + cast(partition_number as nvarchar(5)) +
		' with (data_compression = PAGE)' compression_sql,
	log(rw_ratio/((1.*current_size) / compressed_size))
	from tempdb.dbo.compression_candidates
where (1.*current_size) / compressed_size > 3
order by log(rw_ratio/((1.*current_size) / compressed_size)) desc;
[/code]

Feel free to modify this, but if you do, please let me know because I'd like the improvements.
