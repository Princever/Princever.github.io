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

1. Parallel degree = 1. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..4750277.34 rows=1 width=97) (actual time=59445.496..59445.496 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 1
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4749277.24 rows=1 width=97) (actual time=59443.601..59443.601 rows=0 loops=2)
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

2. Parallel degree = 2. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..4321355.77 rows=1 width=97) (actual time=39887.224..39887.224 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 2
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4320355.67 rows=0 width=97) (actual time=39884.653..39884.653 rows=0 loops=3)
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

3. Parallel degree = 4. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3904689.10 rows=1 width=97) (actual time=24104.645..24104.645 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 4
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3903689.00 rows=0 width=97) (actual time=24101.571..24101.571 rows=0 loops=5)
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

4. Parallel degree = 6. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3696355.77 rows=1 width=97) (actual time=17259.111..17259.111 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 6
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3695355.67 rows=0 width=97) (actual time=17255.757..17255.757 rows=0 loops=7)
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

5. Parallel degree = 8. Data = 3200 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3592189.10 rows=1 width=97) (actual time=13443.716..13443.716 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 8
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3591189.00 rows=0 width=97) (actual time=13440.041..13440.041 rows=0 loops=9)
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

1. Parallel degree = 1. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                       
		                                             
		-----------------------------------------------------------------------------------------------------
		---------------------------------------------
		 Gather  (cost=1000.00..11874192.69 rows=1 width=97) (actual time=221331.238..221331.238 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 1
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..11873192.59 rows=1 width=97) (actual time=221317.556..221317.556 rows=0 loops=2)
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

2. Parallel degree = 2. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                       
	                                             
		-----------------------------------------------------------------------------------------------------
		---------------------------------------------
		 Gather  (cost=1000.00..10801888.77 rows=1 width=97) (actual time=183935.853..183935.853 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 2
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..10800888.67 rows=0 width=97) (actual time=183921.180..183921.180 rows=0 loops=3)
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

3. Parallel degree = 4. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                        
		                                            
		-----------------------------------------------------------------------------------------------------
		--------------------------------------------
		 Gather  (cost=1000.00..9760222.10 rows=1 width=97) (actual time=197697.673..197697.673 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 4
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..9759222.00 rows=0 width=97) (actual time=197601.176..197601.176 rows=0 loops=5)
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

4. Parallel degree = 6. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                        
		                                            
		-----------------------------------------------------------------------------------------------------
		--------------------------------------------
		 Gather  (cost=1000.00..9239388.77 rows=1 width=97) (actual time=165120.498..165120.498 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 6
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..9238388.67 rows=0 width=97) (actual time=165116.993..165116.993 rows=0 loops=7)
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

5. Parallel degree = 8. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                        
		                                            
		-----------------------------------------------------------------------------------------------------
		--------------------------------------------
		 Gather  (cost=1000.00..8978972.10 rows=1 width=97) (actual time=197103.075..197103.075 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 8
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..8977972.00 rows=0 width=97) (actual time=197099.152..197099.152 rows=0 loops=9)
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

6. Parallel degree = 9. Data = 8000 k lines:

		postgres=# EXPLAIN ANALYZE VERBOSE select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                        
		                                            
		-----------------------------------------------------------------------------------------------------
		--------------------------------------------
		 Gather  (cost=1000.00..8892166.54 rows=1 width=97) (actual time=184638.439..184638.439 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 9
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..8891166.44 rows=0 width=97) (actual time=184634.785..184634.785 rows=0 loops=9)
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

#### With Modified Parallel Degree Stratage When Data Quantity = 3200 k lines ####

1. Parallel degree = 0. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                            QUERY PLAN                               
		                             
		-----------------------------------------------------------------------------------------------------
		-----------------------------
		 Seq Scan on public.pgbench_accounts  (cost=0.00..5778689.00 rows=1 width=97) (actual time=117252.556..117252.556 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		   Rows Removed by Filter: 200000000
		   Buffers: shared hit=2144 read=3276545
		 Planning time: 0.097 ms
		 Execution time: 117252.588 ms
		(7 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-0.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-0.svg)

2. Parallel degree = 1. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
	                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..4750277.34 rows=1 width=97) (actual time=59795.382..59795.382 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 1
		   Buffers: shared hit=2229 read=3276513
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4749277.24 rows=1 width=97) (actual time=59761.987..59761.987 rows=0 loops=2)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 100000000
		         Buffers: shared hit=2176 read=3276513
		         Worker 0: actual time=59744.723..59744.723 rows=0 loops=1
		           Buffers: shared hit=1054 read=1643659
		 Planning time: 0.147 ms
		 Execution time: 59796.119 ms
		(13 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-1.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-1.svg)

3. Parallel degree = 2. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..4321355.77 rows=1 width=97) (actual time=39985.548..39985.548 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 2
		   Buffers: shared hit=2346 read=3276449
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..4320355.67 rows=0 width=97) (actual time=39982.959..39982.959 rows=0 loops=3)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 66666667
		         Buffers: shared hit=2240 read=3276449
		         Worker 0: actual time=39981.803..39981.803 rows=0 loops=1
		           Buffers: shared hit=753 read=1095691
		         Worker 1: actual time=39981.821..39981.821 rows=0 loops=1
		           Buffers: shared hit=713 read=1097414
		 Planning time: 0.134 ms
		 Execution time: 39986.251 ms
		(15 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-2.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-2.svg)

4. Parallel degree = 4. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3904689.10 rows=1 width=97) (actual time=24150.869..24150.869 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 4
		   Buffers: shared hit=2548 read=3276353
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3903689.00 rows=0 width=97) (actual time=24147.704..24147.704 rows=0 loops=5)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 40000000
		         Buffers: shared hit=2336 read=3276353
		         Worker 0: actual time=24146.809..24146.809 rows=0 loops=1
		           Buffers: shared hit=457 read=651822
		         Worker 1: actual time=24147.129..24147.129 rows=0 loops=1
		           Buffers: shared hit=457 read=657474
		         Worker 2: actual time=24146.803..24146.803 rows=0 loops=1
		           Buffers: shared hit=455 read=651824
		         Worker 3: actual time=24147.244..24147.244 rows=0 loops=1
		           Buffers: shared hit=469 read=660107
		 Planning time: 0.117 ms
		 Execution time: 24151.620 ms
		(19 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-4.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-4.svg)

5. Parallel degree = 6. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3696355.70 rows=1 width=97) (actual time=17273.312..17273.312 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 6
		   Buffers: shared hit=3454 read=3275553
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3695355.60 rows=0 width=97) (actual time=17269.891..17269.891 rows=0 loops=7)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 28571429
		         Buffers: shared hit=3136 read=3275553
		         Worker 0: actual time=17269.126..17269.126 rows=0 loops=1
		           Buffers: shared hit=440 read=469266
		         Worker 1: actual time=17269.233..17269.233 rows=0 loops=1
		           Buffers: shared hit=441 read=462374
		         Worker 2: actual time=17269.265..17269.265 rows=0 loops=1
		           Buffers: shared hit=449 read=463089
		         Worker 3: actual time=17269.265..17269.265 rows=0 loops=1
		           Buffers: shared hit=443 read=462672
		         Worker 4: actual time=17269.666..17269.666 rows=0 loops=1
		           Buffers: shared hit=443 read=472062
		         Worker 5: actual time=17269.699..17269.699 rows=0 loops=1
		           Buffers: shared hit=444 read=472467
		 Planning time: 0.116 ms
		 Execution time: 17274.088 ms
		(23 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-6.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-6.svg)

6. Parallel degree = 8. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                          
		-----------------------------------------------------------------------------------------------------
		------------------------------------------
		 Gather  (cost=1000.00..3592189.10 rows=1 width=97) (actual time=13469.517..13469.517 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 8
		   Buffers: shared hit=2920 read=3276193
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3591189.00 rows=0 width=97) (actual time=13465.812..13465.812 rows=0 loops=9)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 22222222
		         Buffers: shared hit=2496 read=3276193
		         Worker 0: actual time=13465.122..13465.122 rows=0 loops=1
		           Buffers: shared hit=278 read=364187
		         Worker 1: actual time=13465.059..13465.059 rows=0 loops=1
		           Buffers: shared hit=271 read=362618
		         Worker 2: actual time=13465.323..13465.323 rows=0 loops=1
		           Buffers: shared hit=276 read=365799
		         Worker 3: actual time=13465.278..13465.278 rows=0 loops=1
		           Buffers: shared hit=278 read=363548
		         Worker 4: actual time=13465.582..13465.582 rows=0 loops=1
		           Buffers: shared hit=278 read=365418
		         Worker 5: actual time=13465.286..13465.286 rows=0 loops=1
		           Buffers: shared hit=275 read=363551
		         Worker 6: actual time=13465.761..13465.761 rows=0 loops=1
		           Buffers: shared hit=276 read=366019
		         Worker 7: actual time=13465.774..13465.774 rows=0 loops=1
		           Buffers: shared hit=273 read=363183
		 Planning time: 0.112 ms
		 Execution time: 13470.236 ms
		(27 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-8.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-8.svg)

7. Parallel degree = 12. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                         
		-----------------------------------------------------------------------------------------------------
		-----------------------------------------
		 Gather  (cost=1000.00..3488022.40 rows=1 width=97) (actual time=9857.036..9857.036 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 12
		   Buffers: shared hit=3132 read=3276193
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3487022.30 rows=0 width=97) (actual time=9852.592..9852.592 rows=0 loops=13)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 15384615
		         Buffers: shared hit=2496 read=3276193
		         Worker 0: actual time=9851.568..9851.568 rows=0 loops=1
		           Buffers: shared hit=204 read=266836
		         Worker 1: actual time=9850.689..9850.689 rows=0 loops=1
		           Buffers: shared hit=142 read=177983
		         Worker 2: actual time=9851.913..9851.913 rows=0 loops=1
		           Buffers: shared hit=198 read=266846
		         Worker 3: actual time=9850.961..9850.961 rows=0 loops=1
		           Buffers: shared hit=200 read=257553
		         Worker 4: actual time=9852.290..9852.290 rows=0 loops=1
		           Buffers: shared hit=194 read=266890
		         Worker 5: actual time=9852.381..9852.381 rows=0 loops=1
		           Buffers: shared hit=203 read=266889
		         Worker 6: actual time=9852.576..9852.576 rows=0 loops=1
		           Buffers: shared hit=203 read=238067
		         Worker 7: actual time=9852.662..9852.662 rows=0 loops=1
		           Buffers: shared hit=184 read=254475
		         Worker 8: actual time=9852.612..9852.612 rows=0 loops=1
		           Buffers: shared hit=200 read=266444
		         Worker 9: actual time=9853.134..9853.134 rows=0 loops=1
		           Buffers: shared hit=145 read=221097
		         Worker 10: actual time=9853.157..9853.157 rows=0 loops=1
		           Buffers: shared hit=200 read=260870
		         Worker 11: actual time=9853.154..9853.154 rows=0 loops=1
		           Buffers: shared hit=200 read=262751
		 Planning time: 0.129 ms
		 Execution time: 9858.116 ms
		(35 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-12.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-12.svg)


8. Parallel degree = 16. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                         
		-----------------------------------------------------------------------------------------------------
		-----------------------------------------
		 Gather  (cost=1000.00..3435939.10 rows=1 width=97) (actual time=8866.598..8866.598 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 16
		   Buffers: shared hit=1680 read=3277857
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3434939.00 rows=0 width=97) (actual time=8861.451..8861.451 rows=0 loops=17)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 11764706
		         Buffers: shared hit=832 read=3277857
		         Worker 0: actual time=8859.796..8859.796 rows=0 loops=1
		           Buffers: shared hit=44 read=161957
		         Worker 1: actual time=8859.643..8859.643 rows=0 loops=1
		           Buffers: shared hit=43 read=161631
		         Worker 2: actual time=8860.001..8860.001 rows=0 loops=1
		           Buffers: shared hit=45 read=161391
		         Worker 3: actual time=8860.171..8860.171 rows=0 loops=1
		           Buffers: shared hit=44 read=161996
		         Worker 4: actual time=8860.340..8860.340 rows=0 loops=1
		           Buffers: shared hit=44 read=208941
		         Worker 5: actual time=8860.187..8860.187 rows=0 loops=1
		           Buffers: shared hit=43 read=216364
		         Worker 6: actual time=8861.568..8861.568 rows=0 loops=1
		           Buffers: shared hit=42 read=162752
		         Worker 7: actual time=8861.723..8861.723 rows=0 loops=1
		           Buffers: shared hit=43 read=213945
		         Worker 8: actual time=8861.858..8861.858 rows=0 loops=1
		           Buffers: shared hit=43 read=162878
		         Worker 9: actual time=8861.925..8861.925 rows=0 loops=1
		           Buffers: shared hit=62 read=198814
		         Worker 10: actual time=8862.085..8862.085 rows=0 loops=1
		           Buffers: shared hit=62 read=175208
		         Worker 11: actual time=8861.323..8861.323 rows=0 loops=1
		           Buffers: shared hit=62 read=236455
		         Worker 12: actual time=8861.925..8861.925 rows=0 loops=1
		           Buffers: shared hit=62 read=224491
		         Worker 13: actual time=8862.138..8862.138 rows=0 loops=1
		           Buffers: shared hit=43 read=184521
		         Worker 14: actual time=8861.932..8861.932 rows=0 loops=1
		           Buffers: shared hit=44 read=211549
		         Worker 15: actual time=8861.907..8861.907 rows=0 loops=1
		           Buffers: shared hit=62 read=216003
		 Planning time: 0.109 ms
		 Execution time: 8867.650 ms
		(43 rows)

		postgres=#

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-16.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-16.svg)

9. Parallel degree = 20. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                         
		-----------------------------------------------------------------------------------------------------
		-----------------------------------------
		 Gather  (cost=1000.00..3404689.08 rows=1 width=97) (actual time=8057.978..8057.978 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 20
		   Buffers: shared hit=5092 read=3274657
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3403688.98 rows=0 width=97) (actual time=8052.220..8052.220 rows=0 loops=21)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 9523810
		         Buffers: shared hit=4032 read=3274657
		         Worker 0: actual time=8050.299..8050.299 rows=0 loops=1
		           Buffers: shared hit=182 read=147018
		         Worker 1: actual time=8050.192..8050.192 rows=0 loops=1
		           Buffers: shared hit=181 read=144808
		         Worker 2: actual time=8050.569..8050.569 rows=0 loops=1
		           Buffers: shared hit=177 read=146918
		         Worker 3: actual time=8050.732..8050.732 rows=0 loops=1
		           Buffers: shared hit=183 read=148824
		         Worker 4: actual time=8050.752..8050.752 rows=0 loops=1
		           Buffers: shared hit=179 read=145957
		         Worker 5: actual time=8050.950..8050.950 rows=0 loops=1
		           Buffers: shared hit=182 read=147057
		         Worker 6: actual time=8051.088..8051.088 rows=0 loops=1
		           Buffers: shared hit=178 read=145613
		         Worker 7: actual time=8051.212..8051.212 rows=0 loops=1
		           Buffers: shared hit=184 read=146736
		         Worker 8: actual time=8051.173..8051.173 rows=0 loops=1
		           Buffers: shared hit=176 read=143268
		         Worker 9: actual time=8051.985..8051.985 rows=0 loops=1
		           Buffers: shared hit=171 read=147076
		         Worker 10: actual time=8051.679..8051.679 rows=0 loops=1
		           Buffers: shared hit=178 read=168375
		         Worker 11: actual time=8052.417..8052.417 rows=0 loops=1
		           Buffers: shared hit=252 read=190200
		         Worker 12: actual time=8053.005..8053.005 rows=0 loops=1
		           Buffers: shared hit=190 read=193366
		         Worker 13: actual time=8052.798..8052.798 rows=0 loops=1
		           Buffers: shared hit=196 read=171431
		         Worker 14: actual time=8053.455..8053.455 rows=0 loops=1
		           Buffers: shared hit=185 read=169205
		         Worker 15: actual time=8052.899..8052.899 rows=0 loops=1
		           Buffers: shared hit=179 read=146660
		         Worker 16: actual time=8053.421..8053.421 rows=0 loops=1
		           Buffers: shared hit=182 read=145881
		         Worker 17: actual time=8053.268..8053.268 rows=0 loops=1
		           Buffers: shared hit=179 read=146701
		         Worker 18: actual time=8053.698..8053.698 rows=0 loops=1
		           Buffers: shared hit=249 read=166242
		         Worker 19: actual time=8053.621..8053.621 rows=0 loops=1
		           Buffers: shared hit=249 read=163202
		 Planning time: 0.116 ms
		 Execution time: 8059.505 ms
		(51 rows)
		
		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-20.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-20.svg)

10. Parallel degree = 24. Data = 3200 k lines:

		postgres=# EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';
		                                                                  QUERY PLAN                         
		                                         
		-----------------------------------------------------------------------------------------------------
		-----------------------------------------
		 Gather  (cost=1000.00..3383855.77 rows=1 width=97) (actual time=7847.472..7847.472 rows=0 loops=1)
		   Output: aid, bid, abalance, filler
		   Number of Workers: 24
		   Buffers: shared hit=3448 read=3276513
		   ->  Parallel Seq Scan on public.pgbench_accounts  (cost=0.00..3382855.67 rows=0 width=97) (actual time=7840.068..7840.068 rows=0 loops=25)
		         Output: aid, bid, abalance, filler
		         Filter: (pgbench_accounts.filler = 'foo'::bpchar)
		         Rows Removed by Filter: 8000000
		         Buffers: shared hit=2176 read=3276513
		         Worker 0: actual time=7835.053..7835.053 rows=0 loops=1
		           Buffers: shared hit=97 read=127747
		         Worker 1: actual time=7838.093..7838.093 rows=0 loops=1
		           Buffers: shared hit=89 read=132494
		         Worker 2: actual time=7837.959..7837.959 rows=0 loops=1
		           Buffers: shared hit=91 read=130744
		         Worker 3: actual time=7838.603..7838.603 rows=0 loops=1
		           Buffers: shared hit=88 read=133042
		         Worker 4: actual time=7838.783..7838.783 rows=0 loops=1
		           Buffers: shared hit=89 read=133531
		         Worker 5: actual time=7838.664..7838.664 rows=0 loops=1
		           Buffers: shared hit=91 read=133687
		         Worker 6: actual time=7839.145..7839.145 rows=0 loops=1
		           Buffers: shared hit=91 read=127882
		         Worker 7: actual time=7839.181..7839.181 rows=0 loops=1
		           Buffers: shared hit=91 read=130367
		         Worker 8: actual time=7839.215..7839.215 rows=0 loops=1
		           Buffers: shared hit=89 read=135218
		         Worker 9: actual time=7839.567..7839.567 rows=0 loops=1
		           Buffers: shared hit=90 read=126644
		         Worker 10: actual time=7839.601..7839.601 rows=0 loops=1
		           Buffers: shared hit=90 read=130893
		         Worker 11: actual time=7839.831..7839.831 rows=0 loops=1
		           Buffers: shared hit=90 read=129762
		         Worker 12: actual time=7839.940..7839.940 rows=0 loops=1
		           Buffers: shared hit=91 read=125501
		         Worker 13: actual time=7840.016..7840.016 rows=0 loops=1
		           Buffers: shared read=130324
		         Worker 14: actual time=7839.967..7839.967 rows=0 loops=1
		           Buffers: shared hit=91 read=135244
		         Worker 15: actual time=7840.760..7840.760 rows=0 loops=1
		           Buffers: shared hit=92 read=129374
		         Worker 16: actual time=7841.047..7841.047 rows=0 loops=1
		           Buffers: shared hit=90 read=128296
		         Worker 17: actual time=7841.049..7841.049 rows=0 loops=1
		           Buffers: shared hit=91 read=130529
		         Worker 18: actual time=7841.164..7841.164 rows=0 loops=1
		           Buffers: shared hit=90 read=131568
		         Worker 19: actual time=7841.341..7841.341 rows=0 loops=1
		           Buffers: shared hit=93 read=127192
		         Worker 20: actual time=7841.522..7841.522 rows=0 loops=1
		           Buffers: shared hit=88 read=132989
		         Worker 21: actual time=7841.703..7841.703 rows=0 loops=1
		           Buffers: shared hit=89 read=132046
		         Worker 22: actual time=7841.939..7841.939 rows=0 loops=1
		           Buffers: shared hit=93 read=129622
		         Worker 23: actual time=7841.702..7841.702 rows=0 loops=1
		           Buffers: shared hit=90 read=135927
		 Planning time: 0.121 ms
		 Execution time: 7848.291 ms
		(59 rows)

		postgres=# 

	Flame Graph(<a href="http://princever.github.io/res/images/Test_of_Parallel_Sequence_Scan/2000m-24.svg">detials</a>):
	![]({{ site.baseurl }}/res/images/Test_of_Parallel_Sequence_Scan/2000m-24.svg)


Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!