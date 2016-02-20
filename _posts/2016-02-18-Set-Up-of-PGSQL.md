---  
layout: post
title: Set up PostgreSQL
category: "Database"
---  

Here is the summary of my work during the internship on day 1.

### What To Do ###

The main target is about PostgreSQL, a free and open-source database system.
My work is based on following environment:




	OS: Mac OS 10.11;
	Version: PostgreSQL 9.6devel;

---------------------

### Installation of PostgreSQL ###

The first thing to to is to install pgSQL, first we can get the source code of lastest version by using git:

	$ git clone git://git.postgresql.org/git/postgresql.git
	
Since we have **make** tool installed(On mac OS we should install the **command line tool** first), the only parameter we need to adjust is the path of directory which we will use as the installation directory.
Actually we needn't to change the **configure** file, and the we just attach the directory after our command:

	$ ./configure --prefix=/your/path/to/PostgreSQL/master --enable-debug --enable-cassert --enable-taptest CFLAGS=-O0
	$ make -j 2
	$ make install

Here all the installation step was finished.

### Running PostgreSQL ###

For convenience we add the **bin** directory into environment variables in **~/.bashrc**. Such as:

	export PATH=$PATH:/path/to/your/PostgreSQL/master/bin

Before running we should do the initial step with command.

	$ initdb -D data -E UTF8 --no-locale

And then start the db server with:

	$ pg_ctl start -D $PGDATA

Here **PGDATA** is the directory we store our databases. Which actually mean **/path/to/your/PostgreSQL/master/data** here, we added it into environment variables in **~/.bashrc**:

	export PGDATA=/path/to/your/PostgreSQL/master/data

Similar, we use 

	$ pg_ctl stop -D $PGDATA

to stop server.

### Database SQL ###

Actually all SQL looks the same, so here we won't illustrate them here. The select, update and delete queries. etc. are the same with what we learned in university courses.

Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!