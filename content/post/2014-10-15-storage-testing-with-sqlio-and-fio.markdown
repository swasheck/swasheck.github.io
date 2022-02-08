---
author: swasheck
comments: true
date: 2014-10-15T20:50:48+00:00
layout: post
link: https://swasheck.wordpress.com/2014/10/15/storage-testing-with-sqlio-and-fio/
slug: storage-testing-with-sqlio-and-fio
title: Storage Testing With SQLIO and FIO
wordpress_id: 258
categories: 
- "SQL Server"
tags:
- fio
- sqlio
- storage testing
---

Recently I've been tasked with testing storage for a hardware purchase for a new, large SQL Server implementation. I've not gotten everything "down pat" as far as SQL Server IO methods, but I figured I'd take on the challenge of getting better.
<!-- more -->


### SQLIO: The go-to tool


There are plenty of SQLIO resources on the Internet, but Jonathan Kehayias has a [good post](http://sqlblog.com/blogs/jonathan_kehayias/archive/2010/05/25/parsing-sqlio-output-to-excel-charts-using-regex-in-powershell.aspx) that uses SQLIO well and also references other posts. I set up my tests with a PowerShell script (incorporating Jonathan's RegEx pattern matching) to run 1 to 64 threads (multiples of 2 only), with 1 to 128 outstanding IO requests (also multiples of 2), with 64KB and 512KB block sizes, both random and sequential on a Windows Server 2012 R2 server.

[code language="powershell"]
#RegEx matching courtesy of Jonathan Kehayias' SQLIOResults PowerShell script, available: 
# http://sqlblog.com/blogs/jonathan_kehayias/archive/2010/05/25/parsing-sqlio-output-to-excel-charts-using-regex-in-powershell.aspx

$results = New-Object system.data.datatable;

$results.Columns.add("Threads");
$results.Columns.add("Operation");
$results.Columns.add("IOSize");
$results.Columns.add("IOType");
$results.Columns.Add("PendingIO");
$results.Columns.Add("IOPS");
$results.Columns.Add("MB/s");
$results.Columns.Add("Min_Latency_MS");
$results.Columns.Add("Max_Latency_MS");
$results.Columns.Add("Avg_Latency_MS");

$max_threads = 64;
$max_outstanding_requests = 128;
$block_sizes = @(64,512);
$duration = 30;
$kind = @("R","W");
$factr = @("random","sequential");
#Set-Location C:\SQLIO;

#C:\SQLIO\sqlio.exe -FC:\SQLIO\param.txt -kW -t2 -s15 -dE -o1 -frandom -b64 -BH -LS

foreach ($ki in $kind) 
{    
    $k = "-k$ki";
    
    foreach ($fa in $factr)
    {
        $f = "-f$fa";
        foreach($block_size in $block_sizes)
        {
            $b = "-b$block_size";
            $th = 2;        
            while ($th -le $max_threads)
            {            
                $s = "-s$duration";     
                $t = "-t$th";     
                $or = 1;
                while ($or -le $max_outstanding_requests)
                {
                    $row = $results.NewRow();
                    $o = "-o$or";      
                    #write-host "C:\SQLIO\sqlio.exe -BH -LS $s $k $f $b $t $o";                    
                    $r = E:\SQLIO\sqlio.exe -BH -LS $s $k $f $b $t $o \DB52\testfile.dat;
                    #$i = $r.Split("`n")[10].Split(":")[1].Trim() 
                    $i = [decimal]([regex]::Match($r, "IOs\/sec\:\s+(\d+\.\d+)?").Groups[1].Value);
                    #$m = $r.Split("`n")[11].Split(":")[1].Trim() 
                    $m = [decimal]([regex]::Match($r, "MBs\/sec\:\s+(\d+\.\d+)?").Groups[1].Value);
                    
                    
                    $min_latency = [int]([regex]::Match($r, "Min.{0,}?\:\s(\d+)?").Groups[1].Value)
                    $max_latency = [int]([regex]::Match($r, "Max.{0,}?\:\s(\d+)?").Groups[1].Value)
                    $avg_latency = [int]([regex]::Match($r, "Avg.{0,}?\:\s(\d+)?").Groups[1].Value)                                  

                    "-o " + $or + ", threads: $th kind: $ki blocksize: $block_size $fa " + $i + " iops, " + $m + " MB/sec, " + $l + " ms" + $min_latency + " ms " + $max_latency + " ms " + $avg_latency + " ms";
                    $row["Threads"] = $th;
                    $row["Operation"] = $ki;
                    $row["IOSize"] = $block_size;
                    $row["IOType"] = $fa;
                    $row["PendingIO"] = $or;
                    $row["IOPS"] = $i;
                    $row["MB/s"] = $m;
                    $row["Min_Latency_MS"] = $min_latency;
                    $row["Max_Latency_MS"] = $max_latency;
                    $row["Avg_Latency_MS"] = $avg_latency;
                    $results.Rows.Add($row);
                    $or = $or * 2;                    
                }
                $th = $th * 2;
            }
        }
    }
}

$results | Export-Csv results.csv -notypeinformation
[/code]

An odd thing happened along the way. I started receiving these errors with the 512KB reads and writes at thread counts over 16 and outstanding IOs of 128.
`
init_thread: VirtualAlloc (0x00800000 bytes for I/O Buffer): Not enough storage is available to process this command.
exiting
`
[Apparently, I wasn't the only one having these issues.](https://social.msdn.microsoft.com/Forums/sqlserver/en-US/f1df2670-0add-46b6-b7b5-12daae6d360f/sqlio-on-server-2012?forum=sqltools)

While searching for answers, James Lupolt ([b](http://lupolt.net)|[t](https://twitter.com/jlupoltsql)) pointed me to [some](http://brettroux.blogspot.co.uk/2012/02/compellent-too-clever-for-sqlio.html) [blog](http://blog.delphix.com/uday/2012/09/19/sqlio_fio/) [posts](https://www.simple-talk.com/blogs/2011/06/28/sqlio-writes/) that highlighted the fact that SQLIO actually initializes files with 0x0 - which some have called "zero" but, as Grant Fritchey notes, is actually a "null" character.

[code language="powershell"]
Get-Content testfile.dat
[/code]

confirms:

![gc_sqlio](https://swasheck.files.wordpress.com/2014/10/gc_sqlio.png?w=300)

Many SANs will recognize, and compress, repeating characters and as a result, may artificially inflate your SAN's performance (per the Delphix blog). This would be great if our data files had a large amount of data that could be compressed at the block level. Unfortunately, this doesn't characterize our data files and, as such, we're not benchmarking as accurately as we could.


### IOMeter: Ehrmagerd


Evidently I don't have the IQ to be able to handle IOMeter. I wanted multi-threaded access to files and I couldn't figure it out. In fact, I agree with Brent Ozar wholeheartedly when he says:


<blockquote>I gotta be honest with you: Iometer isn’t the friendliest tool around. I think it makes SQLIO look easy.</blockquote>


I mean, check this out:
[![iometer](https://swasheck.files.wordpress.com/2014/10/iometer.png?w=300)](https://swasheck.files.wordpress.com/2014/10/iometer.png)


### FIO: A Challenger Appears


The Delphix blog referenced an interesting tool - [FIO](http://www.bluestop.org/fio/) which was touted as a more flexible, platform-agnostic storage testing system. I will admit that it took a while to figure out how to make it run, but once I did I found it to be quite useful. I'll spare you the gory details and just get to the part where I figured out how to make it work.

A parameter mapping between the SQLIO and FIO is possible, but not 1:1 so it may be easier to use an actual example of a hardware-buffered, random write, single-threaded, outstanding IO of 1, 90 second, 64KB blocksize test on a 50MB file called testfile.dat:
`
--param.txt
testfile.dat 2 0x0 50

--sqlio.exe test (after having expanded the file)
sqlio.exe -BH -kW -frandom -t1 -o1 -s90 -b64 testfile.dat

-- fio config (after creating a file)
fio.exe --direct=1 -rw=randwrite -numjobs=1 --iodepth=1 --time_based=1 --runtime=30s --blocksize=64k --filename=testfile.dat --size=50m
`
--direct appears to be a direct call to the hardware (which may, or may not be able to satisfy from cache and roughly equivalent to the -BH hardware buffering switch in SQLIO), whereas --buffered would be equivalent to the software buffering option in SQLIO.

There are also a number of other helpful switches for FIO, including --output which will let you specify the test result location, --ioengine (which engine to use), --group_reporting will report on the group results instead of the job results (so you can get how each thread performed if you'd like), and --loops will allow you to run the test multiple times.

Since the big issue was the null character values in the SQLIO test file, I wanted to make sure that we have actual data in the file:[![fiofile](https://swasheck.files.wordpress.com/2014/10/fiofile.png?w=300)](https://swasheck.files.wordpress.com/2014/10/fiofile.png)

My comparable FIO testing PowerShell script to the above SQLIO script looks something like this:

[code language="powershell"]
#threads
#op type
#duration
#io size
#pending io

$max_threads = 64;
$max_outstanding_requests = 128;
$block_sizes = @(64,512);
$duration = 30;
$rws = @("read","randread","write","randwrite");
#Set-Location \\your\path\to\fio\here
.\fio.exe --size=50m --iodepth=1 --blocksize=8k --filename=50mtest.dat --name=build --thread;
foreach($rw in $rws)
{
    $rw;
    foreach($bs in $block_sizes)
    {
        $th = 1;
        while($th -le $max_threads)
        {
            $iodepth = 1;
            while ($iodepth -le $max_outstanding_requests)
            {
                $bl_size = "--blocksize=" + $bs + "k";
                write-host "--rw=$rw --size=50m --iodepth=$iodepth --direct=1 $bl_size --nrfiles=1 --numjobs=$th --group_reporting=1 --ioengine=windowsaio --loops=1 --time_based=1 --runtime=30s --thread --filename=50mtest.dat --name=test --output=G:\fio_tests\testrun_50mb-$rw-$bs-$th-$iodepth.txt";
                .\fio.exe --rw=$rw --size=50m --iodepth=$iodepth --direct=1 $bl_size --nrfiles=1 --numjobs=$th --group_reporting=1 --ioengine=windowsaio --loops=1 --time_based=1 --runtime=30s --thread --filename=50mtest.dat --name=test --output=G:\fio_tests\testrun_50mb-$rw-$bs-$th-$iodepth.txt;
                $iodepth = $iodepth * 2;                
            }
            $th = $th * 2;
        }        
    }
}
[/code]

Tomorrow I'll submit a follow-up post to show how I parsed my FIO results and we can compare the difference between the null and randomized data tests to see if it truly made a difference.
