---
title: "Temporary Statistics"
description: ""
date: 2017-04-04T00:00:00
tags:
- "statistics"
- "temporary statistics"
date: "2017-04-06"
categories: 
- "SQL Server"
---

I am speaking on statistics in June at the [Denver SQL Server User Group](https://denversql.org) but didn't really want to rehash much of the same old information that I've heard in the past. There's a wealth of information about how to interpret stat headers, density vectors, and histograms. There's also been plenty of virtual ink spilled on when statistics are updated (if you have auto update enabled).

One new(ish, to me) and interesting topic I stumbled across is the notion of temporary statistics. These are special statistics objects for a database that is participating in an AlwaysOn Availability Group, but located on a secondary server that is available for Read-Intent connections. Theoretically, if you're using a readable secondary then you're trying to shed at least some of your read workload from the primary. That workload would ostensibly carry a different workload and query planning could require access to statistics that are either missing or stale on the primary. To accomodate this situation, the SQL Server team creates temporary statistics that exist in tempdb for the read-only database.

There exists [documentation](https://docs.microsoft.com/en-us/sql/database-engine/availability-groups/windows/active-secondaries-readable-secondary-replicas-always-on-availability-groups#Read-OnlyStats) for statistics on read-only databases and Sunil Agarwal has [a couple](https://blogs.msdn.microsoft.com/sqlserverstorageengine/2011/12/22/alwayson-challenges-with-statistics-on-readonly-database-database-snapshot-and-secondary-replica/) of [good posts](https://blogs.msdn.microsoft.com/sqlserverstorageengine/2011/12/22/alwayson-making-latest-statistics-available-on-readable-secondary-read-only-database-and-database-snapshot/) that help to shed a bit more light on the topic. While most of my questions were technically already covered in these links, they were still topics that I wanted to examine for myself. For the purposes of these tests I'm using a dataset that I created from [USGS Earthquake data](https://earthquake.usgs.gov/earthquakes/search/).


**Test 1:** *Automatically Create Statistics*

Statistics are created when the SQL Server engine recognizes a need for valid statistics that do not currently exist on a column, and when the database is configured to automatically create statistics (the default setting is "on"). As noted in the linked documents, statistics will be created on a readable secondary if statistics do not currently exist. On my clean `[earthquake]` database I run my favorite statistics-information-gathering query to identify statistics on the secondary database:

```
use earthquake;
go
select 
	schema_name = sh.name, 
	table_name = t.name, 
	stat_name = s.name, 
	stats_columns = stuff ((
							select ',' + c.name colname
							from sys.stats_columns sc
							join sys.columns c 
								on c.column_id = sc.column_id
									and c.object_id = sc.object_id
							where sc.object_id = s.object_id
								and sc.stats_id = s.stats_id
							order by sc.stats_column_id
							for xml path(''),type
						).value('.','nvarchar(max)')
						,1,1,''),
	s.is_temporary,
	spi.last_updated, 
	spi.rows, 
	spi.unfiltered_rows, 
	spi.rows_sampled,
	spi.modification_counter, 
	spi.steps	
from sys.stats s
join sys.tables t 
	on t.object_id = s.object_id
join sys.schemas sh
	on sh.schema_id = t.schema_id
cross apply sys.dm_db_stats_properties_internal(s.object_id,s.stats_id) spi;
```

and see these nice statistics appear. 
![prestats](https://swasheck.gitlab.io/fixed/2017-04-01-stats_pre.png)

If I run a query that forces statistics creation (earthquakes with a magnitude of at least 3.0 that occurred on a holiday and generated a 'Red' alert):
```
select 	
	d.Date, 
	t.Time,
	e.EventTitle, 
	e.Magnitude,
	e.Alert
from dbo.Event e 
join dbo.Times t 
	on e.EventTimeID = t.TimeID
join dbo.Dates d 
	on e.EventDateID = d.DateID
where 
	e.Magnitude >= 3
	and e.Alert = 'red'
	and d.IsHoliday = 1
order by d.Date,t.Time
option (recompile);
```
and see these nice temporary statistics appear. 

![prestats](https://swasheck.gitlab.io/fixed/2017-04-01-temp_stats.png)

The documentation didn't lie. We have temporary statistics and they all have `_readonly_database_statistics` appended to their names. 

**Test 2:** *Updating Autocreated Temporary Statistics*

Having successfully created temporary statistics, we can now explore their update patterns. 

A manual statistics update attempt
```
update statistics dbo.event(_WA_Sys_00000008_0F975522_readonly_database_statistics) with fullscan
```
results in:
```
Msg 3906, Level 16, State 13, Line 133
Failed to update database "earthquake" because the database is read-only.
```
That's to be expected, based on the documentation. Additionally, per the documentation, I can drop temporary statistics
```
drop statistics dbo.Event._WA_Sys_00000004_108B795B_readonly_database_statistics; --EventDateID
```
```
Command(s) completed successfully.
```

**Test 2.2:** *More Updating Autocreated Temporary Statistics*

A column on the primary takes enough changes to trigger a stats update, but no statistics are present on the primary. However, an auto-created, temporary statistic exists on the readable secondary. What happens?

First we make changes to the primary:

```
update dbo.Event
	set Magnitude = Magnitude+10
```
![mod](https://swasheck.gitlab.io/fixed/2017-04-01-mod_stats.png)

We see that modifications *are* reflected on the secondary so now we set about running our query again. Because, in my mind, there is the very real possibility that *if* the statistics are updated it's a drop and recreate behind the scenes, I decided to watch the activity with the `auto_stats` Extended Event session.

```
create event session [stat_load] on server 
add event sqlserver.auto_stats(
    action(sqlserver.sql_text)
    where ([database_id]=(10)))
    add target package0.event_file(set filename=N'stat_load')
    with (event_retention_mode=allow_single_event_loss,track_causality=on)
go
```

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-updated_stats.png)

Upon completion of the earlier earthquake query, we can look at the statistics again and see that the statistic has a more recent last update time than the others. Additionally, the statistic that we dropped was also recreated with roughly the same time. The `auto_stats` event session is helpful in this situation.

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-auto_stats.png)

What we see is that the statistic on the `[Magnitude]` column (red box) was, in fact, updated, while the statistic on `[EventDateID]` column (blue box) was created. If the former statistic was simply dropped and re-created, we would expect to see the same "Created:" sequence for that statistic as well. 


**Test 3:** *Overlapping AutoCreated Statistics*

Assume the statistics created in Test 2 are present on the secondary and the primary takes a read that creates an analogous statistic on the primary. We already know, based on the documentation, that statistic will be shipped to the secondary, so we will have a duplicate statistic on the secondary. We also already know that the newer statistic will be used in any future query planning on the secondary. However, are there any differences in the salient metrics for each of these autostats?

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-primary_auto.png)

As expected, statistics are automatically created on the primary and appear on the secondary. Additionally, we see that they have the same sample rate as both the primary statistic, and the temporary statistics created earlier.

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-primary_secondary.png)

If we look at the statistics objects' information, we see identical (except for time) stat headers and density vectors. Additionally, the histograms are identical.

```
dbcc show_statistics('dbo.Event',_WA_Sys_00000008_108B795B_readonly_database_statistics) with histogram
dbcc show_statistics('dbo.Event',_WA_Sys_00000008_108B795B) with histogram
```
![mod](https://swasheck.gitlab.io/fixed/2017-04-01-histogram.png)

**Test 4:** *Stale Statistics on the Primary, Read Workload on the Secondary*

Let's assume that there exists a statistics object on a column on both the primary and secondary replicas of the Availability Group. The primary replica takes changes on that column, but does not have a read workload that would require loading that statistic for compiling a plan. However, the secondary replica *does* take reads on that column. 

Once again, let's make a change to the `[Magnitude]` column:
```
update dbo.Event
	set Magnitude = Magnitude-10
```
and then look at the status of the statistics

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-test4.png)

Then we run our read query workload on the secondary and check our statistics.

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-test4.2.png)

Whereas Microsoft documentation led me to believe that there would simply be a new, temporary, read-only statistics object created, what actually happened was that the statistics object on the secondary was updated, but is now marked as a temporary statistic. What happens when we decide to manually failover the Availability Group?

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-test4.3.png)

The statistic is not updated, but it changes from a temporary statistic to a permanent statistic. However, this conversion does not hold true for a temporary statistic that is created on the secondary. After a failover, this statistic object

![mod](https://swasheck.gitlab.io/fixed/2017-04-01-test5.png)

is removed and any read workloads that run against that column on the replica that is now the primary will require a newly created statistics object.



***CONCLUSION***

Temporary statistics on your readable secondary are an interesting feature that allow for better query planning for read-only workloads than otherwise would be available with the only the stats shipped from the primary. Stored in tempdb, these statistics objects are created and updated when the workload requires such behavior, independent of any action on the primary. Of course, if extant statistics are updated on the primary, those updates will be replayed on the secondary and provide a "fresh" statistics object to any read-only workload that may require it. Finally, temporary statistics that are created on the secondary will be deleted upon failover of the Availability Group. 