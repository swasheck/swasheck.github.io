---
author: swasheck
comments: true
date: 2013-12-14T04:24:05+00:00
layout: post
link: https://swasheck.wordpress.com/2013/12/13/stats-health/
slug: stats-health
title: Stats Health
categories:
- "SQL Server"
---

I had the good fortune of attending [Kimberly Tripp's data skew presentation at the PASS Summit 2013 ](http://www.sqlpass.org/summit/2013/Sessions/SessionDetails.aspx?sid=5050)in Charlotte, NC back in October. In this presentation she revealed some code that she'd developed to help analyze skew in your data and make suggestions for filtered stats. This inspired me to take a look at statistics in some of my servers. 
<!-- more -->
So I downloaded her presentation demo and tried to get a sense for what she was doing. Overall I think I understood it, until she analyzed for skew. I couldn't really grasp what she was doing (and I put that more on me than on her - she's much smarter and more experienced than I am). However, I still needed to analyze my stats for skew. My calculation is much more basic than hers - I look at the RANGE_ROWS and do some statistical measures for outliers. Yes, I ran stats on stats.

![xzibit](http://cdn.memegenerator.net/instances/500x/43850131.jpg)

I decided that I would also flag old/stale stats and also flag stats where the row count on the stats didn't agree with the row count on the object. These are configurable parameters.

So here it is, my "stats_health" query. I've posted it on [GitHub ](https://github.com/swasheck/sql_server_stats_health)and I welcome checkouts and updates.

Please note that there is zero documentation below the header, and that there is also zero error handling. Finally,<del> it also doesn't handle user-defined data types very gracefully either</del> it handles some user-defined types, but not hierarchyid, geography, and geometry based types. This is a tool that I have begun using and wanted to share it. I will add these things (or you can too) as time permits.
