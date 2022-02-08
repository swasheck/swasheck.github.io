---
author: swasheck
comments: true
date: 2014-01-13T23:23:11+00:00
layout: post
link: https://swasheck.wordpress.com/2014/01/13/tracking-sql-server-and-sqlsat-271-abq/
slug: tracking-sql-server-and-sqlsat-271-abq
title: '"Tracking" SQL Server (and SQLSat #271 - ABQ)'
categories:
- "SQL Server"
---

**
Update
I’ve noticed some range and scale issues so I took out the explicit schema creation (for now). For right now it just selects into a table, dropping the table if it already exists.
**

Back in October I had the opportunity to attend Jonathan Kehayias'[ "SQL Server Archaeology"](http://www.sqlpass.org/summit/2013/Sessions/SessionDetails.aspx?sid=4606) presentation. I'd been trying to wrap my brain around Extended Events for a while, but his presentation got me motivated to dive into the system_health Event Session. 
<!-- more -->
Additionally, I also decided to submit a presentation idea to [SQL Saturday #271](http://www.sqlsaturday.com/271/eventhome.aspx) (Albuquerque, NM) titled, "Extended Events In Real Life." The whole point of this presentation is to take some examples of ways that I've used Extended Events to solve problems or gain insight into the processes on SQL Server. I'm pretty excited (and a bit nervous) about this opportunity to speak. 

To this end, I've also been given the opportunity to present a portion of "XEIRL" at our local [SQL Server User Group](http://denversql.org/) this Thursday. This has accelerated both the nerves and the preparation. Since it's been a while since I've posted anything, I figured I'd post the demo scripts that I am going to be using in this user group presentation. 

Let's say that someone comes to you and says "I wanted to tell you that 'the database' was slow yesterday but never got a chance because my keyboard was abducted by aliens, my legs were sent off to NCAR for analysis of their impact on barometric pressure, and I was using my phone to get past level 239 of Candy Crush." What do you do? Hopefully you have a good monitoring tool (if you don't, I recommend [SQL Sentry Performance Advisor](http://sqlsentry.net/performance-advisor/sql-server-performance.asp)) and you can get an idea of what happened. If you don't, or if you're like me and just like to figure out "internals" stuff, and you're on SQL Server 2012, then it could be system_health to the rescue. 

This session has been created by the product team to help troubleshoot potential performance issues on the database server. System_health _does_ exist in SQL Server 2008, but XEs were fairly new and seem to be a somewhat incomplete implementation making that version inadequate. Most of our environment is 2012 so I've been in luck with a more complete implementation of system_health. [Read up on it here](http://msdn.microsoft.com/en-us/library/ff877955(sql.110).aspx). In order to maintain some sembalence of brevity, let's cut to the chase. You want to see what happened. In the linked file (below), you can see the setup for how I handle the system_health data. I take the XML data from the event file and dump it into a temp table and then process the XML from this temp table. My brain thinks in terms of manageable chunks and this fit the bill.

Now, what happened?
One of the beautiful things about the 2012 version of system_health is the addition of sp_server_diagnostics output. This gives a snapshot of query processing, IO, system, and resource "health" (if you run the procedure independent of the event session you'll get a few more components) at the moment of execution. So first, let's look for any of these components that may be unhealthy:
[code language="sql"]
select d.grp,a.* 
from [system_health].[system_health_spsd] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		);
[/code]

Using this as our base, we can even trend waits (I chose non-preemptive waits by count, but the same logic applies for any combination of preemptive and by duration waits) to get a better look at what was going on:
[code language="sql"]


-- QUERY PROCESSING IN WARNING STATE

select d.grp,a.* 
from [system_health].[system_health_spsd] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)

-- GET EVERYTHING SURROUNDING QP IN WARNING STATE
select d.grp,a.* 
from [system_health].[system_health_xlat] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)
and event_name <> 'security_error_ring_buffer_recorded'

-- WAITS ANALYSIS

SELECT 
	grp, 
	event_sequence,
	spsd_component_state,
	event_timestamp,
	wait_type,
	waits, 	
	waits - LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) as wait_delta,
	CASE
		WHEN waits - LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) = waits
			AND LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NEW WAIT'
		WHEN waits - LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) > 0
			THEN 'INCREASE'
		WHEN waits - LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) < 0
			THEN 'DECREASE'
		WHEN waits - LAG(waits,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NO CHANGE'
	END as wait_delta_desc,
	avg_wait_time, 
	avg_wait_time - LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) as avg_wait_time_delta,
	CASE
		WHEN avg_wait_time - LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = avg_wait_time
			AND LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NEW WAIT'
		WHEN avg_wait_time - LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) > 0
			THEN 'INCREASE'
		WHEN avg_wait_time - LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) < 0
			THEN 'DECREASE'
		WHEN avg_wait_time - LAG(avg_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NO CHANGE'
	END as avg_wait_time_delta_desc,
	max_wait_time, 
	max_wait_time - LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) as avg_wait_time_delta,
	CASE
		WHEN max_wait_time - LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = max_wait_time
			AND LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NEW WAIT'
		WHEN max_wait_time - LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) > 0
			THEN 'INCREASE'
		WHEN max_wait_time - LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) < 0
			THEN 'DECREASE'
		WHEN max_wait_time - LAG(max_wait_time,1,0) over (partition by grp,wait_type order by event_sequence) = 0
			THEN 'NO CHANGE'
	END as avg_wait_time_delta_desc
FROM (

		select 
			d.grp, 
			DENSE_RANK() OVER (PARTITION BY d.grp ORDER BY a.event_timestamp) as event_sequence,
			a.spsd_component,
			a.spsd_component_state,
			a.event_timestamp,
			bc.n.value('@waitType','sysname') wait_type,
			bc.n.value('@waits','bigint') waits,	
			bc.n.value('@averageWaitTime','bigint') avg_wait_time,	
			bc.n.value('@maxWaitTime','bigint') max_wait_time,	
			event_data
		from [system_health].[system_health_spsd] a
		join (
			select	
				ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
				event_timestamp 
			from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
		) d
		on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
				and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
		OUTER APPLY npwaits_bycount.nodes('/byCount/wait') bc(n)
		where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
			and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
				and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
				)
			and a.spsd_component = 'QUERY_PROCESSING'
	) agg
	order by grp,event_sequence

-- DEEPER DIVE INTO WAITS, WHERE THERE ANY RECORDED?
-- SOS_WORKER: Waiting for a thread
-- LCK_M_S: Typical locking/blocking
select d.grp,a.* 
from [system_health].[system_health_waits] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)		

--GO BACK AND CONFIRM THAT THERE WAS A BLOCKED PROCESS
select d.grp,a.* 
from [system_health].[system_health_spsd] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)

-- CHECK FOR ANY ERRORS
select d.grp,a.* 
from [system_health].[system_health_errors] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)


-- CHECK FOR DEADLOCKS
select d.grp,a.* 
from [system_health].[system_health_deadlocks] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)

-- FINALLY, CHECK THE RINGBUFFERS
select d.grp,a.* 
from [system_health].[system_health_ring_buffers] a
join (
	select	
		ROW_NUMBER() OVER (ORDER BY event_timestamp) as grp,
		event_timestamp 
	from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
) d
on a.event_timestamp <= DATEADD(MINUTE,6,d.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,d.event_timestamp)
where exists (select 1 from [system_health].[system_health_xlat] b where b.spsd_component_state <> 'CLEAN'
	and a.event_timestamp <= DATEADD(MINUTE,6,b.event_timestamp)
		and a.event_timestamp >= DATEADD(MINUTE,-6,b.event_timestamp)
		)
		-- I ELIMINATE security_error_ring_buffer_recorded api_name='ImpersonateSecurityContext' because
		--	it's noisy and is only tellingvyou that it's still alive checking for security errors
	and event_name <> 'security_error_ring_buffer_recorded'
[/code]

On and on and on you can go to further plumb the depths of what was happening at that time. You can look for deadlocks, errors, and analyze the output of ring buffers to really get a good idea of what was happening. I've [attached](https://app.box.com/s/c32yqtqef9ep109th2vy) all scripts so that you can have a good time seeing what you can get out of your system_health event session. 

**
Update
I've noticed some range and scale issues so I took out the explicit schema creation (for now). For right now it just selects into a table, dropping the table if it already exists.
 
