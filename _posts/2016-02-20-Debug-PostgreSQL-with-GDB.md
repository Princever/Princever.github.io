---  
layout: post
title: Debug PostgreSQL with GDB
category: "Database"
---  

Now let's talking about debuging postgreSQL with GUN GDB.

First we need to be familiar with the basic structure of processes of postgreSQL. Which is shown as the following fugure:




![]({{ site.baseurl }}/res/images/gdbpgsql/1.png)

There is a permanent process postgres that is called 'postmaster' running at backend listening to its ports and deal with all connections.

When users connects to the server, this process deals with the request and accessto  the db engine for information.



Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!