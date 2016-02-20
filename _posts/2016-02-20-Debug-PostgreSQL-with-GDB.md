---  
layout: post
title: Debug PostgreSQL with GDB
category: "Database"
---  

Now let's talking about debuging postgreSQL with GUN GDB.
First we need to be familiar with the basic structure of processes of postgreSQL. Which is shown as the following fugure:




![]({{ site.baseurl }}/res/images/gdbpgsql/1.png)

There is a permanent process postgres that is called 'postmaster' running at backend listening to its ports and deal with all connections.When users connects to the server, this process deals with the request and access to the db engine for information.


### Install GDB And Create a Certificate ###

It is hard to trace codes with our eyes in a giant system like postgreSQL, so we can use GDB, a simple tool for debug. As my platform is on a mac, so we should install GDB with homebrew first:

	$ brew install gdb

Because of the secure system of the new kernal of mac, GDB(one process) is not to allowed to control another process without permission of root. So we may make a signature for our GBD. This can be done in the **Keychain** function. So we just create a certificate as following step:

1. Create a new Certificate.
![]({{ site.baseurl }}/res/images/gdbpgsql/2.png)

2. Name the certificate and select type as **Code Signing**.
![]({{ site.baseurl }}/res/images/gdbpgsql/3.png)

3. Click continue until the specifying of location to store this certificate, select **System**.
![]({{ site.baseurl }}/res/images/gdbpgsql/4.png)

4. Now the certificate has been established.
![]({{ site.baseurl }}/res/images/gdbpgsql/5.png)

5. Let's select **Always Trust** in `Get Info → Trust → Code Signing` when right-clicking.
![]({{ site.baseurl }}/res/images/gdbpgsql/6.png)

Then we `cd` into the directory of gdb and attach the code sign with gdb.

	codesign -s gdb-cert gdb

After a reboot, the certificate is already avaliable.

### Debug PostgreSQL With GDB ###

First we start the database with order:

	$ pg_ctl start -D $PGDATA

Then we login in as the default user:

	PrinceMacbook:~ Prince$ psql postgres
	psql (9.6devel)
	Type "help" for help.

	postgres=# 

Then we open another termial and find the process id of the postgreSQL.

	$ ps x|grep post

and we can find the idle process with id of 793.

	599   ??  Ss     0:00.00 postgres: checkpointer process     
	600   ??  Ss     0:00.05 postgres: writer process     
	601   ??  Ss     0:00.01 postgres: wal writer process     
	602   ??  Ss     0:00.01 postgres: autovacuum launcher process     
	603   ??  Ss     0:00.01 postgres: stats collector process     
	793   ??  Ss     0:00.00 postgres: Prince postgres [local] idle  
	597 s000  S      0:00.03 /Applications/PostgreSQL/pgsql/master/bin/postgres -D /Applications/PostgreSQL/pgsql/master/data
	792 s000  S+     0:00.01 psql postgres
	985 s002  R+     0:00.00 grep post

Then we use `gdb postgres 6150` to debug.

	PrinceMacbook:~ Prince$ gdb postgres 793
	GNU gdb (GDB) 7.10
	Copyright (C) 2015 Free Software Foundation, Inc.
	License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
	This is free software: you are free to change and redistribute it.
	There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
	and "show warranty" for details.
	This GDB was configured as "x86_64-apple-darwin15.3.0".
	Type "show configuration" for configuration details.
	For bug reporting instructions, please see:
	<http://www.gnu.org/software/gdb/bugs/>.
	Find the GDB manual and other documentation resources online at:
	<http://www.gnu.org/software/gdb/documentation/>.
	For help, type "help".
	Type "apropos word" to search for commands related to "word"...
	Reading symbols from postgres...done.
	Attaching to program: /Applications/PostgreSQL/pgsql/master/bin/postgres, process 793
	0x00007fff8f12039e in poll () from /usr/lib/system/libsystem_kernel.dylib
	(gdb) 

Here we add a breakpoint at function **ExecResult** with b command.

	(gdb) b ExecResult
	Breakpoint 1 at 0x10e12f8ac: file nodeResult.c, line 75.
	(gdb) 

Then we type a query, for example, `select 1;`. The termial will pause and output nothing after we input the command because a breakpoint is set in process **postgres**. So we should use **c** command for continuing the process.

	(gdb) c
	Continuing.

	Breakpoint 1, ExecResult (node=0x7fbdc90a6d50) at nodeResult.c:75
	75		econtext = node->ps.ps_ExprContext;
	(gdb) 

From above outputed information we can find the running stats of code. And we can also use **bt** to find all function call paths before **ExecResult**.

	(gdb) bt
	#0  ExecResult (node=0x7fbdc90a6d50) at nodeResult.c:75
	#1  0x000000010e0ff450 in ExecProcNode (node=0x7fbdc90a6d50)
    	at execProcnode.c:392
	#2  0x000000010e0f9616 in ExecutePlan (estate=0x7fbdc90a6c38, 
    	planstate=0x7fbdc90a6d50, use_parallel_mode=0 '\000', 
    	operation=CMD_SELECT, sendTuples=1 '\001', numberTuples=0, 
    	direction=ForwardScanDirection, dest=0x7fbdc9076480) at execMain.c:1566
	#3  0x000000010e0f9508 in standard_ExecutorRun (queryDesc=0x7fbdc9008038, 
    	direction=ForwardScanDirection, count=0) at execMain.c:338
	#4  0x000000010e0f930d in ExecutorRun (queryDesc=0x7fbdc9008038, 
    	direction=ForwardScanDirection, count=0) at execMain.c:286
	#5  0x000000010e2d50b3 in PortalRunSelect (portal=0x7fbdc9816838, 
    	forward=1 '\001', count=0, dest=0x7fbdc9076480) at pquery.c:942
	#6  0x000000010e2d4a8a in PortalRun (portal=0x7fbdc9816838, 
    	count=9223372036854775807, isTopLevel=1 '\001', dest=0x7fbdc9076480, 
    	altdest=0x7fbdc9076480, completionTag=0x7fff51d8aea0 "") at pquery.c:786
	#7  0x000000010e2d01fd in exec_simple_query (
    	query_string=0x7fbdc9074a38 "select 1;") at postgres.c:1094
	#8  0x000000010e2cf4fe in PostgresMain (argc=1, argv=0x7fbdc90034e0, 
    	dbname=0x7fbdc9003340 "postgres", username=0x7fbdc9003320 "Prince")
    	at postgres.c:4021
	#9  0x000000010e22e5e9 in BackendRun (port=0x7fbdc8d00800) at postmaster.c:4258
	#10 0x000000010e22d829 in BackendStartup (port=0x7fbdc8d00800)  at postmaster.c:3932
	#11 0x000000010e22c82f in ServerLoop () at postmaster.c:1690
	#12 0x000000010e229ffb in PostmasterMain (argc=3, argv=0x7fbdc8c03c20) at postmaster.c:1298
	#13 0x000000010e160a70 in main (argc=3, argv=0x7fbdc8c03c20) at main.c:223
	(gdb) 

The function below called the function above. For example, **ExecProcNode** in **#1** called **ExecResult** in **#0**, and itself is called by **ExecutePlan** in **#2**. Now let's pay attention to **#7**.

	#7  0x000000010e2d01fd in exec_simple_query (
    	query_string=0x7fbdc9074a38 "select 1;") at postgres.c:1094

It is where **select 1** was processed.

We can also use **list** to view codes around the line are now being executed.

	(gdb) list
	70		TupleTableSlot *resultSlot;
	71		PlanState  *outerPlan;
	72		ExprContext *econtext;
	73		ExprDoneCond isDone;
	74	
	75		econtext = node->ps.ps_ExprContext;
	76	
	77		/*
	78		 * check constant qualifications like (2 > 1), if not already done
	79		 */
	(gdb) 

Using **up** command to go upwards from the callee **ExecResult** to the caller **ExecProcNode**. With **list** command we can ensure the actual place where ExecResult been called.

	(gdb) up
	#1  0x000000010e0ff450 in ExecProcNode (node=0x7fbdc90a6d50) at execProcnode.c:392
	392				result = ExecResult((ResultState *) node);
	(gdb) list
	387		{
	388				/*
	389				 * control nodes
	390				 */
	391			case T_ResultState:
	392				result = ExecResult((ResultState *) node);
	393				break;
	394	
	395			case T_ModifyTableState:
	396				result = ExecModifyTable((ModifyTableState *) node);
	(gdb) 

Similarly, we can use **down** command to go downwards from caller to callee.

	(gdb) down
	#0  ExecResult (node=0x7fbdcb003550) at nodeResult.c:75
	75		econtext = node->ps.ps_ExprContext;
	(gdb) 

Finally we used **c** command again, and the execution of query will continue and some output appeared.

	postgres=# select 1;
 	?column? 
	----------
	        1
	(1 row)

	postgres=# 

As for another termial, we use **Quit** to quit gdb at the next breakpoint.

	(gdb) c
	Continuing.

	Breakpoint 1, ExecResult (node=0x7fbdcb003150) at nodeResult.c:75
	75		econtext = node->ps.ps_ExprContext;
	(gdb) Quit
	A debugging session is active.

		Inferior 1 [process 793] will be detached.

	Quit anyway? (y or n) y
	Detaching from program: /Applications/PostgreSQL/pgsql/master/bin/postgres, process 793

That's all about a simple introduction of debuging postgresSQL with GUN GDB.

Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!