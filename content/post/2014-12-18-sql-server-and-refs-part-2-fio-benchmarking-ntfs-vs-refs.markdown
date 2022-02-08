---
author: swasheck
comments: true
date: 2014-12-18T07:25:39+00:00
layout: post
link: https://swasheck.wordpress.com/2014/12/18/sql-server-and-refs-part-2-fio-benchmarking-ntfs-vs-refs/
slug: sql-server-and-refs-part-2-fio-benchmarking-ntfs-vs-refs
title: 'SQL Server and ReFS: Part 2 - FIO Benchmarking NTFS vs. ReFS'
wordpress_id: 303
categories: 
- "SQL Server"
---

As I was sitting in Bob Ward’s Inside SQL Server I/O presentation, something interesting caught my eye on a slide that noted that ReFS is now supported for SQL Server 2014. I'd run into problems with 2012 so I'd just given up but this looks promising. I am neither a filesystem aficionado, nor a dilettante but I know that there are some interesting features of ReFS at which Windows server admins are looking to see if it’s viable.

[Previously](http://swasheck.wordpress.com/2014/11/11/sql-server-and-refs-part-1-dbcc-and-in-memory-oltp/), I examined a couple of features that were potential blockers for 2014/ReFS adoption: DBCC CHECKDB and In-Memory OLTP support.

So with it established that these issues have been addressed and that SQL Server 2014 does support ReFS, should we consider it as a viable option? Let's see from a FIO benchmark perspective.
<!-- more -->



### Test Harness



OS: Windows Server 2012 R2 Enterprise Edition
RAM: 8GB 
CPU: 2 Dual-Core AMD @ 2.93 GHz
Storage: Pure FA-420

I recently wrote [a post](http://swasheck.wordpress.com/2014/10/15/storage-testing-with-sqlio-and-fio/) where I discussed using FIO instead of SQLIO to do storage testing. In it there are scripts that I used to set up my storage tests. For completeness, here is a script:
```
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
```



## Configurations of Note




### File Size


I also created a second script that changed the 50m (50MB data file) value to 300g (300GB). 50MB is small enough to reside in the cache of most SAN controller cache, while 300GB would be large enough to test the actual storage itself. I did this because I wanted to measure the influence of each file system over the performance of the middle layer (storage controller, HBA card, etc.) and the storage itself. 


### IO (Block) Size


I chose three common IO size patterns for SQL Server:
8KB: Common Read/Write
64KB: "Ramp-Up" Read (in which SQL Server will read an extent) as well as the largest log write
512KB: Largest possible read read (including read ahead)



### Threads


Threads represent concurrent accesses to the data file(s). The general guidance is to test up to the number of Logical CPUs present on your test harness ([Jose Barreto](http://blogs.technet.com/b/josebda/archive/2013/03/28/sqlio-powershell-and-storage-performance-measuring-iops-throughput-and-latency-for-both-local-disks-and-smb-file-shares.aspx)) with the rationale being that "SQL Server only exposes one worker thread per logical CPU at a time" ([SQL Meditation](http://blogs.msdn.com/b/sqlmeditation/archive/2013/04/04/choosing-what-sqlio-tests-to-run-and-automating-sqlio-testing-somewhat.aspx)). You'll note that I went well past this (all the way up to 64 threads) as my test harness only had 4 logical CPUs. I did this just to see how the system responded to the pressure (it didn't crash, so there's that) but in order to get a good sense of how ReFS performed against NTFS, we'll need to focus on the 4-thread results.



### Pending IO


SQL Sasquatch has an [excellent post](http://sql-sasquatch.blogspot.com/2014/05/windows-wait-queue-emulex-fc-hba.html) that helps understand Pending IO in that it exposes how the OS and hardware fill services queues with requests. From a storage testing perspective, a "pending IO" is an in-flight IO that is in one of the queues. Since the default queue depth on most HBAs is 32 any number of pending IO requests will (at least initially) go directly to the service queue to be serviced by the storage subsystem. Anything greater than 32 will begin to fill the wait queue. At this point, it's important to highlight two important details:



	
  1. The queue depth is per HBA card. My test system has 2 cards each with a queue depth of 32, so I will have up to 64 total IOs in the service queue at any given moment


	
  2. The number of pending IOs in the configuration is _per thread_. Therefore, 4 threads submitting 32 pending IOs will result in 128 total IOs to be serviced.





### Test Results


The screen shots are hard to see, which is why I've provided a link to the source Excel workbook for you to view, if you wish.


#### IOPS


[![8K_4T_IOPS](https://swasheck.files.wordpress.com/2014/12/8k_4t_iops.png)](https://swasheck.files.wordpress.com/2014/12/8k_4t_iops.png)

Starting with 8K blocksize operations shows a fairly consistent increase and plateau for almost every combination of filesystem and operation with the exceptions of ReFS random reads which peak at 32 pending IO and then decline. The other exception is the sequential write under ReFS which essentially remains flat throughout the test.

[![64K_4T_IOPS](https://swasheck.files.wordpress.com/2014/12/64k_4t_iops1.png)](https://swasheck.files.wordpress.com/2014/12/64k_4t_iops1.png)

The specific capture is IOPS responding to pending IO at 4 threads doing 64K blocksize IOs on the 300GB file. We see that until we get to 128 pending IO, all file systems are fairly consistent, however NTFS outperforms ReFS across all points. 

[![512K_4T_IOPS](https://swasheck.files.wordpress.com/2014/12/512k_4t_iops1.png)](https://swasheck.files.wordpress.com/2014/12/512k_4t_iops1.png)

Bumping to 512K blocksize operations shows drops in IOPS after 8 outstanding IOs for most file system operations except random and sequential reads for NTFS, and sequential reads for ReFS. Random reads for ReFS consistently drops until 128 pending IO where it inexplicably jumps back up. 



#### Throughput (MB/s)



[![8K_4T_MBS](https://swasheck.files.wordpress.com/2014/12/8k_4t_mbs.png)](https://swasheck.files.wordpress.com/2014/12/8k_4t_mbs.png)

Throughput (focusing on the storage controller) at 8K reads shows a fairly consistent pattern of plateauing at 16 pending IO (again, with the exception of ReFS sequential writes). 

[![64K_4T_MBS](https://swasheck.files.wordpress.com/2014/12/64k_4t_mbs.png)](https://swasheck.files.wordpress.com/2014/12/64k_4t_mbs.png)

At 64K blocksizes, NTFS writes suffer over against their peers. This is notable because at this and the 512K block size, ReFS writes generally outperform their NTFS peers. However, this wouldn't really come into play because, other than backups, I'm not familiar with a SQL Server write pattern that would exceed 64K.

[![512K_4T_MBS](https://swasheck.files.wordpress.com/2014/12/512k_4t_mbs1.png)](https://swasheck.files.wordpress.com/2014/12/512k_4t_mbs1.png)

Because of the above information about no 512K writes, I filtered out the writes for the 512K blocksizes (I also did this for the IOPS chart, but the result was less dramatic). NTFS still outperforms its ReFS peer operations.



### Summary


I'd read that with integrity streams disabled, ReFS suffers a significant penalty and, for the most part that holds true. This leads to NTFS operations still consistently outperforming peer ReFS operations. ReFS does generally scale similarly to NTFS, plateauing, peaking, and declining at roughly the same configurations, but with consistently lower performance metrics. In short, I'm not recommending using ReFS with SQL Server just yet, even though it's supported.

Feel free to open the workbook below and click around and contribute any other assessments or observations. 
[office src="https://onedrive.live.com/embed?cid=43452EB0C7E41198&resid=43452EB0C7E41198%2120937&authkey=ACcEJyui9ATugOw&em=2&AllowTyping=True&wdHideGridlines=True&wdHideHeaders=True&wdDownloadButton=True" width="1024" height="600"]
