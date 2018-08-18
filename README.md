

Run parsetrc.pl example

       $ ./parsetrc.pl  mytrace.trc

The formatted trace file output looks like:


	----------------------------------------------------------------------------
                    Summary of Activity in trace file
	----------------------------------------------------------------------------
           event                               count total secs    avg ms
        1) ELAPSED                                          921
        2) CPU                                              485
        3) db file sequential read             76259        201     2.644
        4) db file scattered read               7196         20     2.849
        5) SQL*Net message from client            22          0    12.123
        6) asynch descriptor resize           186982          0     0.001
     
	----------------------------------------------------------------------------
	Histogram of latencies  for:                  db file sequential read
	----------------------------------------------------------------------------
     0   64u   .1m   .2m   .5m    1m    2m    4m    8m   16m   33m   65m  .1s .1s+
     0  29239  6676   972  8044   735  1177  5377 17642  6062   286    13  36
	----------------------------------------------------------------------------

	----------------------------------------------------------------------------
	Histogram of latencies by I/O size in # of blocks for:
	                            db file scattered read
	                            direct path read
	                            direct path read temp
	----------------------------------------------------------------------------
	----------------------------------------------------------------------------
	db file scattered read
	      0    64u   .1m   .2m   .5m    1m    2m    4m    8m   16m   33m   65m  .1s .1s+
	 32   0      0     0     0     0     0  5106   702   661   660    50
	  2   0      2     3     0     0     0     0     0     0     4     0
	----------------------------------------------------------------------------

	----------------------------------------------------------------------------
	Breakdown by SQL Statement
	----------------------------------------------------------------------------

	----------------------------------------------------------------------------
	UPDATE TOTO.TRANSACTIONS T1 SET
	      T1.YIELD_YTM = (SELECT NVL(T1.YIELD_YTM,NVL(T2."Reported_Yield",T1.YIELD_YTM))
	    -------------------------------------------------------------
	          events for quyery
	    -------------------------------------------------------------
	    event                               count total secs    avg ms
	    ------------------------------ ---------- ---------- ---------
	    db file sequential read              3349         18     5.493
	    db file scattered read                  1          0    11.417
	    Disk file operations I/O                1          0     0.083
	    -------------------------------------------------------------
	        execution plan for quyery
	    -------------------------------------------------------------
	         'UPDATE  TOTO (cr=5171 pr=3378 pw=0 time=18564002 us)'
	           'FILTER  (cr=5171 pr=3378 pw=0 time=18563998 us)'
	             'NESTED LOOPS  (cr=5171 pr=3378 pw=0 time=18563995 us)'
	               'NESTED LOOPS  (cr=1931 pr=370 pw=0 time=78074 us cost=1264
	                 'SORT UNIQUE (cr=30 pr=30 pw=0 time=30635 us cost=17 
	                   'TABLE ACCESS FULL VATRADE_SUPPORT (cr=30 pr=30 pw=0 time=22918 
	                 'INDEX RANGE SCAN TRADE_ID_SOURCE_IDX (cr=1901 pr=340 pw=0
	               'TABLE ACCESS BY INDEX ROWID LOT_TRANSACTIONS (cr=3240 pr=3008 pw=0
	           'TABLE ACCESS FULL VATRADE_SUPPORT (cr=0 pr=0 pw=0 time=0 us cost=17 
	----------------------------------------------------------------------------
	
The above output has 3 basic sections

    summary of activity 
    histogram of latency
    SQL execution plan


Summary of Activity
The �summary of activity� section is a breakdown of how the time is spent in the trace file.
The most important statistic to compare is the column �total sec.�
 If there are any rows here that show significantly more wait time on devops, then that issue should be addressed.
There are basically 3 types of events in the Summary section


        CPU and Elapsed
        Non-I/O wait events such as lock, latch, space allocation
        IO wait events

1. The CPU and Elapsed can vary for a number of reasons. Elapsed includes idle time, so if the session being traced was idle for different amounts of time then the two traces can have different Elapsed time. The CPU should be relatively the same if the boxes are the same type.
2. The other wait events should account for roughly the same amount of time. The counts can be different and the events can be different, but if there is some event that takes up much more time in one trace than the other trace then that can point to some problem or configuration difference between the databases. For example if a lock shows up in one and not the other, then in one case there is a blocking session slowing down this trace, where as if the other trace has no locks then it was not blocked by another session. In this case there would be an application problem in one trace and not the other.

Comparing the count columns might look like
----------------------------------------------------------------------------

	                       prodb| devops      
	-------------------------------|--------
	           event        count  |  count 
	db file sequential read  9451  |  76259  
	db file scattered read     17  |   7196 

----------------------------------------------------------------------------

Which would mean that the devops read more data than the prodb thus there is something different.  This difference should be addressed before looking deeper into the issue.
The rows to compare are I/O, and the possible I/O rows are:
read I/O

        db file sequential read
        db file scattered read
        direct path read
        direct path read temp

The count for these events should be roughly the same. If the count is higher for devops then virtual is doing one the following:

    1. reading more data � customer has added or change data
    2. has less data cached in the Oracle buffer cache -
        * either the buffer cache is smaller
           *  see init.ora parameter db_block_buffers
           *  or run command �show sga� in sqlplus and look at value �Database Buffers�
        * or the buffer cache has has yet to cache the data because one of the following
           *  first time query has been run
           *  no other query retrieving the relevant data has been run
           *  other queries have forced out the relevant data by  filling the buffer cache with data unrelated to this query
    3. has a different execution plan � see � SQL execution plan�  below

If the counts are roughly the same on prodb and devops but the �avg ms�, ie the average time waited per event, is higher then look at the histogram of latency. See below
write I/O

            direct path write
            direct path write temp
            log file sync

The count for these events should be roughly the same. If the count is higher for one, then the devops is doing a different workload:

direct path write � query is inserting more data
direct path write temp � query is sorting more data or the memory the user is allowed to use for sorting is smaller
log file sync � query/job is committing more.
histogram of latency

If the list of events and count of events are roughly the same  in �summary  of activity� section yet the read I/O events showed significantly more time on devops,  then histogram section will give more detail about these I/O read latency. The goal of the latency histogram is primarily to see if virtual and prodb were using the same amount of host file system cache. There is no general method currently of seeing which I/O came from file system cache or from SAN cache or actually from prodb disk, but given the limits of current hardware one can make some strong inferences from the latency distribution. Latency under 64 microseconds is generally going to be coming from local file system cache. (on 4Gb FC it takes 20us just to transfer an 8K block not accounting for any code stack, scheduling or memory or disk access).

Here is an example comparing prodb to devops for single block reads, which Oracle calls �db file sequential read�

	----------------------------------------------------------------------------
	Histogram of latencies  for:
                                 db file sequential read
	----------------------------------------------------------------------------
	     0   64u    .1m  .2m   .5m    1m   2m   4m    8m   16m   33m  65m  .1s .1s+
	devops
	     0      1    14  908  13238 6900 9197 15603  9056   265   26   12   12
	Proddb
     0   7132   391  118  22189 2794 1688  2003 11969 14877 2003  105    3   3  

(The first column �0? is for any for any reads which reported no latency. This column should have 0 occurrences.)

The column �64u� is the 64 microsecond bucket. On prodb there were 7132 reads where as on  devops there was on 1  indicating that the prodb was benefiting from more reads from the host file system cache.
One should be able to rectify this difference by re-running the query again on the devops database. A second running of the query should benefit from the data cached in the file system cache by the first execution. Other issues which will affect how much data gets cached in the file system cache are

    amount of memory on the machine
    amount of memory allocated to all processes
    activity of processes on the host

The second part the latency histogram is for I/Os that read multiple blocks. The latency for these reads is broke out by the size of the I/O as well as the time. The amount of I/O read in a multiblock read can vary for a number of reasons, but the bulk of I/O sizes should be the same and the max I/O size for each type of I/O should be the same.
SQL execution plan

The next section gives details on each SQL statement in the trace file. There maybe be multiple SQL statements. Some of the SQL statements will have been explicitly run and others may have been kicked of by Oracle to look up relevant information.

	----------------------------------------------------------------------------
	Breakdown by SQL Statement
	----------------------------------------------------------------------------

The statements are ordered by elapsed time with the slowest statements listed first.Find compare each of the statements that took  a significant amount of time and compare the execution plans of these statements. For example if the two two equivalent SQL statements had the following SQL execution plans

	UPDATE  TOTO                                UPDATE  TOTO
	   FILTER                                            FILTER  
	     NESTED LOOPS                                      NESTED LOOPS  
	       NESTED LOOPS                                        NESTED LOOPS  
	         SORT UNIQUE                                          INDEX RANGE SCAN TOTO_IDX 
	           TABLE ACCESS FULL SUPPORT                          SORT UNIQUE   
	         INDEX RANGE SCAN TOTO_IDX                    TABLE ACCESS FULL SUPPORT 
	       TABLE ACCESS BY INDEX ROWID TOTO            TABLE ACCESS BY INDEX ROWID TOTO
	   TABLE ACCESS FULL SUPPORT                   TABLE ACCESS FULL SUPPORT 
	

Then the execution plans woudl be different because the order or the lines in the execution plan are different.The Oracle optimizer has changed the execution plan because of :

        different statistics on the underlying tables
        missing or new indexes on the tables
        different Oracle configuration parameters such as

            db_file_multiblock_read_count
            optimizer_index_cost_adj
            optimizer_index_caching
            or others

