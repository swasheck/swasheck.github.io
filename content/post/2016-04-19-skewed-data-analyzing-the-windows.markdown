---
author: swasheck
comments: true
date: 2016-04-19T18:35:52+00:00
link: https://swasheck.wordpress.com/2016/04/19/skewed-data-analyzing-the-windows/
slug: skewed-data-analyzing-the-windows
title: 'Skewed Data: Analyzing the Windows'
wordpress_id: 623
categories: 
- "SQL Server"
tags:
- skew
- statistics
---

[Previously](https://swasheck.wordpress.com/2016/04/06/skewed-data-finding-the-columns/), we looked at analyzing a table to see which columns in that table may contain skewed data.

That was a good start, but now it's time to look at the statistics that exist on that column to see if we can identify potential candidates for filtered statistics, based on the "windows" between histogram steps.

Much of the logic is the same in this script, except it counts _every_ value in the column. Additionally, it will look at all statistics that exist on that column. If there are no statistics on that column, then we can't do a histogram step window analysis anyway. The same general principles exist for this analysis as well. We're looking at the test statistic (zG1) to determine how skewed the data may actually be.

Just like with the table analysis, I worked around data type issues by using `dense_rank()` over the keys. In the histogram dump table, I created a column called `alt_key` which I then update based on matching the count table key to the histogram step key.

<pre>
<code>
set @sql = 'dbcc show_statistics('''+quotename(@SchemaName)+'.'+quotename(@TableName)+''','''+@StatName+''') 
    with histogram,no_infomsgs;'
print @sql;     
insert into tempdb.dbo.histo (
                        range_hi_key, 
                        range_rows , 
                        eq_rows , 
                        distinct_range_rows ,
                        avg_range_rows )
execute sp_executesql @sql
set @sql = 
        'update h
            set h.actual_eq_rows = c.f,
                h.alt_key = c.x
        from tempdb.dbo.histo h
        join '+(@TallyTable)+' c 
            on h.range_hi_key = c.[key]';

        exec sp_executesql @sql;
</code>
</pre>

What this lets me do is pull the analysis results later, without having to muck with sorting on different data types:


<pre>
<code>
select an.*,confirm_query = 'select * from ' + @TallyTable + ' where ' +
case when cols.last_alt_key is not null 
    then ' where x >= ' + cast(cols.last_alt_key as nvarchar(255)) + ' and ' 
    else ' x <=' + cast(cols.alt_key as nvarchar(255))
    end  + ' order by [key]'
from tempdb.dbo.histo an
join (
        select 
            stat_name,
            last_range_hi_key = lag(range_hi_key,1,null) over (partition by stat_name order by range_hi_key), 
            range_hi_key,
            last_alt_key = lag(alt_key,1,null) over (partition by stat_name order by range_hi_key), 
            alt_key
        from tempdb.dbo.histo
    ) cols
on an.stat_name = cols.stat_name
    and an.range_hi_key = cols.range_hi_key
where an.actual_distinct_range_rows >= 100
order by abs(zg1) desc;
</code>
</pre>

With the above analysis available to us, we'd run the text in `confirm_query` to examine the window (each histogram step is included) to sanity check the analysis and ensure that the window does present with skew and has enough distinct values to make a filtered statistic worthwhile. Please note that all of the normal considerations with regard to filtered statistics apply. They don't update with their filtered threshold is met, but only when the threshold for the entire table is met, and only then if the filtered statistics is loaded for plan (re)compilation. They may add more time to any maintenance task. They may never be used. etc. etc. etc.

[Stats Skew Analysis Script](https://gitlab.com/swasheck/statistics-scripts/blob/65e6fff5a357dd68acdcf47b7b8d4d1b5be33dac/stats%20skew%20analysis.sql)
