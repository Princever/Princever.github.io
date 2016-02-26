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


Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!