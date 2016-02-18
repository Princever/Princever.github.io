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
	
Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!