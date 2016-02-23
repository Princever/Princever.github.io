---  
layout: post
title: Test of Parallel Sequence Scan
category: "Database"
---  

We tried to test the performance of **parallel seq scan**, a new feature in version 9.6, postgreSQL.

### Output Some Logs ###

To record, or, to get access to know about time information of a process, we need to output some logs. We implemented this by adding a few lines in the source code.
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

Here we just all a few lines like:

	struct timeval start,end; 

	/*
	 * About time
	 */

	gettimeofday(&start, NULL );

	elog(LOG, "start time=%f\n", ( 1000000 * start.tv_sec  + start.tv_usec) / 1000000.0 );

	...

	gettimeofday(&end, NULL );

	elog(LOG, "end time=%f\n", ( 1000000 * end.tv_sec  + end.tv_usec) / 1000000.0 );

	long timeuse =1000000 * ( end.tv_sec - start.tv_sec ) + end.tv_usec - start.tv_usec;

	elog(LOG, "time=%f\n", timeuse /1000000.0 );

Then we do `make` and `make install` again, restart the database and then able to find our output.

	PrinceMacbook:postgres Prince$ pgstart
	server starting
	PrinceMacbook:postgres Prince$ LOG:  database system was shut down at 2016-02-24 00:41:08 JST
	LOG:  MultiXact member wraparound protections are now enabled
	LOG:  database system is ready to accept connections
	LOG:  autovacuum launcher started

	PrinceMacbook:postgres Prince$ psql postgres
	psql (9.6devel)
	Type "help" for help.

	postgres=# set client_min_messages='notice';
	LOG:  start time=1456242163.189879
	 
	LOG:  end time=1456242163.190244
		
	LOG:  time=0.000365
	
	SET
	postgres=# select 1;
	LOG:  start time=1456242173.349366
	
	LOG:  end time=1456242173.350068
	
	LOG:  time=0.000702
	
	 ?column? 
	----------
	        1
	(1 row)

	postgres=#

We used level **LOG** here to output infos.