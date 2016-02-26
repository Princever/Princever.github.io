---  
layout: post
title: Test of Parallel Sequence Scan
category: "Database"
---  

We tried to test the performance of **parallel seq scan**, a new feature in version 9.6, postgreSQL.




### Output Some Logs ###

To record, or, to get access to know about run time information of a process, we need to output some logs. We implemented this by adding a few lines in the source code.
**elog** is a kind of function who outputs logs with string information. In the source code, it looks like:

	#ifdef HAVE__BUILTIN_CONSTANT_P
	#define ereport_domain(elevel, domain, rest)	\
		do { \
			if (errstart(elevel, __FILE__, __LINE__, PG_FUNCNAME_MACRO, domain)) \
				errfinish rest; \
			if (__builtin_constant_p(elevel) && (elevel) >= ERROR) \
				pg_unreachable(); \
		} while(0)
	#else							/* !HAVE__BUILTIN_CONSTANT_P */
	#define ereport_domain(elevel, domain, rest)	\
		do { \
			const int elevel_ = (elevel); \
			if (errstart(elevel_, __FILE__, __LINE__, PG_FUNCNAME_MACRO, domain)) \
			errfinish rest; \
			if (elevel_ >= ERROR) \
				pg_unreachable(); \
		} while(0)
	#endif   /* HAVE__BUILTIN_CONSTANT_P */

The elog has several levels as shown in source code:

	/* Error level codes */
	#define DEBUG5		10			/* Debugging messages, in categories of
									 * decreasing detail. */
	#define DEBUG4		11
	#define DEBUG3		12
	#define DEBUG2		13
	#define DEBUG1		14			/* used by GUC debug_* variables */
	#define LOG			15			/* Server operational messages; sent only to
									 * server log by default. */
	#define COMMERROR	16			/* Client communication problems; same as LOG
									 * for server reporting, but never sent to
									 * client. */
	#define INFO		17			/* Messages specifically requested by user (eg
									 * VACUUM VERBOSE output); always sent to
									 * client regardless of client_min_messages,
									 * but by default not sent to server log. */
	#define NOTICE		18			/* Helpful messages to users about query
									 * operation; sent to client and not to server
									 * log by default. */
	#define WARNING		19			/* Warnings.  NOTICE is for expected messages
									 * like implicit sequence creation by SERIAL.
									 * WARNING is for unexpected messages. */
	#define ERROR		20			/* user error - abort transaction; return to
									 * known state */
	/* Save ERROR value in PGERROR so it can be restored when Win32 includes
	 * modify it.  We have to use a constant rather than ERROR because macros
	 * are expanded only when referenced outside macros.
	 */
	#ifdef WIN32
	#define PGERROR		20
	#endif
	#define FATAL		21			/* fatal error - abort process */
	#define PANIC		22			/* take down the other backends with me */

Here we just added a few lines like:

	struct timeval start,end; 

	/*
	 * About time
	 */

	gettimeofday(&start, NULL );

	elog(LOG, "In exec_simple_query(), start time=%f\n", ( 1000000 * start.tv_sec  + start.tv_usec) / 1000000.0 );

	...

	gettimeofday(&end, NULL );

	elog(LOG, "In exec_simple_query(), end time=%f\n", ( 1000000 * end.tv_sec  + end.tv_usec) / 1000000.0 );

	long timeuse =1000000 * ( end.tv_sec - start.tv_sec ) + end.tv_usec - start.tv_usec;

	elog(LOG, "In exec_simple_query(), running time=%f\n", timeuse /1000000.0 );

Then we did `make` and `make install` again, restarted the database and then were able to find our outputs.

	PrinceMacbook:postgres Prince$ pgstart
	server starting
	PrinceMacbook:postgres Prince$ LOG:  database system was shut down at 2016-02-24 00:41:08 JST
	LOG:  MultiXact member wraparound protections are now enabled
	LOG:  database system is ready to accept connections
	LOG:  autovacuum launcher started

	PrinceMacbook:postgres Prince$ psql postgres
	psql (9.6devel)
	Type "help" for help.

	postgres=# select 1;
	LOG:  In exec_simple_query(), start time=1456242173.349366
	
	LOG:  In exec_simple_query(), end time=1456242173.350068
	
	LOG:  In exec_simple_query(), running time=0.000702
	
	 ?column? 
	----------
	        1
	(1 row)

	postgres=#

We used level **LOG** here to output information. If each log repeated twice, remember to do `set client_min_messages='notice'` when connecting to the database.

### Generate Flame Graph With Perf ###

With perf already installed, we `cd` into its directory, run the command:

	# perf record -p pid -F 99 -ag -- sleep time

For a server, the postgresSQL should not be ran with the root account, and perf should not be ran as not the root account. It means we should open two ssh terminals.

Using `ps aux|grep post` finding the process, then we can use perf.

	root@devpg03 FlameGraph]# perf record -p 11097 -F 99 -ag -- sleep 10
	Warning:
	PID/TID switch overriding SYSTEM[ perf record: Woken up 1 times to write data ]
	[ perf record: Captured and wrote 0.206 MB perf.data (~9006 samples) ]

After this we use following commands to output the results into flame graph.

	[root@devpg03 FlameGraph]# perf script | ./stackcollapse-perf.pl > out.perf-folded
	[root@devpg03 FlameGraph]# cat out.perf-folded | ./flamegraph.pl > perf-test2.svg

Also we can use `# perf report --stdio` to view the result in terminal.

Flame graphs shown as following:

+ When quantity of data is 16k lines(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/1.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/1.svg)

+ When quantity of data is 1600k lines(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2.svg)

### Some More Experiments ###

#### Data Quantity = 3200 k lines ####

+ Parallel degree = 1. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..4750277.34 rows=1 width=97) (actual time=59445.496..59445.496 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 1
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4749277.24 rows=1 width=97) (actual 
		time=59443.601..59443.601 rows=0 loops=2)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 100000000
		         Worker 0: actual time=59441.983..59441.983 rows=0 loops=1
		 Planning time: 0.114 ms
		 Execution time: 59446.175 ms
		(10 rows)
		
		postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000-1.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000-1.svg)

+ Parallel degree = 2. Data = 3200 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                  QUERY PLAN                         
	                                          
	-----------------------------------------------------------------------------------------------------
	------------------------------------------
	 Gather  (cost=1000.00..4321355.77 rows=1 width=97) (actual time=39887.224..39887.224 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 2
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4320355.67 rows=0 width=97) (actual 
	time=39884.653..39884.653 rows=0 loops=3)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 66666667
	         Worker 0: actual time=39883.543..39883.543 rows=0 loops=1
	         Worker 1: actual time=39883.479..39883.479 rows=0 loops=1
	 Planning time: 0.150 ms
	 Execution time: 39888.029 ms
	(11 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000-2.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000-2.svg)

+ Parallel degree = 4. Data = 3200 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                  QUERY PLAN                         
	                                          
	-----------------------------------------------------------------------------------------------------
	------------------------------------------
	 Gather  (cost=1000.00..3904689.10 rows=1 width=97) (actual time=24104.645..24104.645 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 4
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3903689.00 rows=0 width=97) (actual 
	time=24101.571..24101.571 rows=0 loops=5)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 40000000
	         Worker 0: actual time=24100.815..24100.815 rows=0 loops=1
	         Worker 1: actual time=24100.554..24100.554 rows=0 loops=1
	         Worker 2: actual time=24101.115..24101.115 rows=0 loops=1
	         Worker 3: actual time=24101.099..24101.099 rows=0 loops=1
	 Planning time: 0.135 ms
	 Execution time: 24105.401 ms
	(13 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000-4.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000-4.svg)

+ Parallel degree = 6. Data = 3200 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                  QUERY PLAN                         
	                                          
	-----------------------------------------------------------------------------------------------------
	------------------------------------------
	 Gather  (cost=1000.00..3696355.77 rows=1 width=97) (actual time=17259.111..17259.111 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 6
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3695355.67 rows=0 width=97) (actual 
	time=17255.757..17255.757 rows=0 loops=7)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 28571429
	         Worker 0: actual time=17255.033..17255.033 rows=0 loops=1
	         Worker 1: actual time=17255.098..17255.098 rows=0 loops=1
	         Worker 2: actual time=17255.280..17255.280 rows=0 loops=1
	         Worker 3: actual time=17255.360..17255.360 rows=0 loops=1
	         Worker 4: actual time=17255.085..17255.085 rows=0 loops=1
	         Worker 5: actual time=17255.669..17255.669 rows=0 loops=1
	 Planning time: 0.129 ms
	 Execution time: 17259.983 ms
	(15 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000-6.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000-6.svg)

+ Parallel degree = 8. Data = 3200 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                  QUERY PLAN                         
	                                          
	-----------------------------------------------------------------------------------------------------
	------------------------------------------
	 Gather  (cost=1000.00..3592189.10 rows=1 width=97) (actual time=13443.716..13443.716 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 8
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3591189.00 rows=0 width=97) (actual 
	time=13440.041..13440.041 rows=0 loops=9)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 22222222
	         Worker 0: actual time=13439.271..13439.271 rows=0 loops=1
	         Worker 1: actual time=13439.353..13439.353 rows=0 loops=1
	         Worker 2: actual time=13439.470..13439.470 rows=0 loops=1
	         Worker 3: actual time=13439.444..13439.444 rows=0 loops=1
	         Worker 4: actual time=13439.750..13439.750 rows=0 loops=1
	         Worker 5: actual time=13439.452..13439.452 rows=0 loops=1
	         Worker 6: actual time=13440.105..13440.105 rows=0 loops=1
	         Worker 7: actual time=13440.176..13440.176 rows=0 loops=1
	 Planning time: 0.131 ms
	 Execution time: 13444.516 ms
	(17 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000-8.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000-8.svg)

#### Data Quantity = 8000 k lines ####

+ Parallel degree = 1. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                    QUERY PLAN                       
	                                             
	-----------------------------------------------------------------------------------------------------
	---------------------------------------------
	 Gather  (cost=1000.00..11874192.69 rows=1 width=97) (actual time=221331.238..221331.238 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 1
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..11873192.59 rows=1 width=97) (actual
	 time=221317.556..221317.556 rows=0 loops=2)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 250000000
	         Worker 0: actual time=221304.152..221304.152 rows=0 loops=1
	 Planning time: 0.128 ms
	 Execution time: 221332.119 ms
	(10 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-1.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-1.svg)

+ Parallel degree = 2. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                    QUERY PLAN                       
	                                             
	-----------------------------------------------------------------------------------------------------
	---------------------------------------------
	 Gather  (cost=1000.00..10801888.77 rows=1 width=97) (actual time=183935.853..183935.853 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 2
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..10800888.67 rows=0 width=97) (actual
	 time=183921.180..183921.180 rows=0 loops=3)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 166666667
	         Worker 0: actual time=183913.963..183913.963 rows=0 loops=1
	         Worker 1: actual time=183914.011..183914.011 rows=0 loops=1
	 Planning time: 0.131 ms
	 Execution time: 183936.582 ms
	(11 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-2.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-2.svg)

+ Parallel degree = 4. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                   QUERY PLAN                        
	                                            
	-----------------------------------------------------------------------------------------------------
	--------------------------------------------
	 Gather  (cost=1000.00..9760222.10 rows=1 width=97) (actual time=197697.673..197697.673 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 4
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..9759222.00 rows=0 width=97) (actual 
	time=197601.176..197601.176 rows=0 loops=5)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 100000000
	         Worker 0: actual time=197577.221..197577.221 rows=0 loops=1
	         Worker 1: actual time=197577.049..197577.049 rows=0 loops=1
	         Worker 2: actual time=197577.225..197577.225 rows=0 loops=1
	         Worker 3: actual time=197577.053..197577.053 rows=0 loops=1
	 Planning time: 0.126 ms
	 Execution time: 197698.441 ms
	(13 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-4.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-4.svg)

+ Parallel degree = 6. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                   QUERY PLAN                        
	                                            
	-----------------------------------------------------------------------------------------------------
	--------------------------------------------
	 Gather  (cost=1000.00..9239388.77 rows=1 width=97) (actual time=165120.498..165120.498 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 6
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..9238388.67 rows=0 width=97) (actual 
	time=165116.993..165116.993 rows=0 loops=7)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 71428571
	         Worker 0: actual time=165116.325..165116.325 rows=0 loops=1
	         Worker 1: actual time=165116.468..165116.468 rows=0 loops=1
	         Worker 2: actual time=165116.339..165116.339 rows=0 loops=1
	         Worker 3: actual time=165116.561..165116.561 rows=0 loops=1
	         Worker 4: actual time=165116.355..165116.355 rows=0 loops=1
	         Worker 5: actual time=165116.774..165116.774 rows=0 loops=1
	 Planning time: 0.140 ms
	 Execution time: 165121.249 ms
	(15 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-6.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-6.svg)

+ Parallel degree = 8. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                   QUERY PLAN                        
	                                            
	-----------------------------------------------------------------------------------------------------
	--------------------------------------------
	 Gather  (cost=1000.00..8978972.10 rows=1 width=97) (actual time=197103.075..197103.075 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 8
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..8977972.00 rows=0 width=97) (actual 
	time=197099.152..197099.152 rows=0 loops=9)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 55555556
	         Worker 0: actual time=197098.497..197098.497 rows=0 loops=1
	         Worker 1: actual time=197098.762..197098.762 rows=0 loops=1
	         Worker 2: actual time=197097.869..197097.869 rows=0 loops=1
	         Worker 3: actual time=197098.808..197098.808 rows=0 loops=1
	         Worker 4: actual time=197098.897..197098.897 rows=0 loops=1
	         Worker 5: actual time=197098.836..197098.836 rows=0 loops=1
	         Worker 6: actual time=197098.960..197098.960 rows=0 loops=1
	         Worker 7: actual time=197099.059..197099.059 rows=0 loops=1
	 Planning time: 0.144 ms
	 Execution time: 197104.030 ms
	(17 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-8.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-8.svg)

+ Parallel degree = 9. Data = 8000 k lines:

	postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
	                                                                   QUERY PLAN                        
	                                            
	-----------------------------------------------------------------------------------------------------
	--------------------------------------------
	 Gather  (cost=1000.00..8892166.54 rows=1 width=97) (actual time=184638.439..184638.439 rows=0 loops=1)
	   Output: aid, bid, abalance, filler
	   Number of Workers: 9
	   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..8891166.44 rows=0 width=97) (actual 
	time=184634.785..184634.785 rows=0 loops=9)
	         Output: aid, bid, abalance, filler
	         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
	         Rows Removed by Filter: 55555556
	         Worker 0: actual time=184634.033..184634.033 rows=0 loops=1
	         Worker 1: actual time=184634.131..184634.131 rows=0 loops=1
	         Worker 2: actual time=184634.282..184634.282 rows=0 loops=1
	         Worker 3: actual time=184634.408..184634.408 rows=0 loops=1
	         Worker 4: actual time=184634.380..184634.380 rows=0 loops=1
	         Worker 5: actual time=184634.596..184634.596 rows=0 loops=1
	         Worker 6: actual time=184634.379..184634.379 rows=0 loops=1
	         Worker 7: actual time=184634.814..184634.814 rows=0 loops=1
	 Planning time: 0.134 ms
	 Execution time: 184639.392 ms
	(17 rows)

	postgres=# 

Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/5000-9.svg">detials</a>):
![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/5000-9.svg)



Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!