---  
layout: post
title: Debug PostgreSQL with GDB
category: "Database"
---  

Now let's talking about debuging postgreSQL with GUN GDB.
First we need to be familiar with the basic structure of processes of postgreSQL. Which is shown as the following fugure:




![]({{ site.baseurl }}/res/images/gdbpgsql/1.png)

There is a permanent process postgres that is called 'postmaster' running at backend listening to its ports and deal with all connections.When users connects to the server, this process deals with the request and access to the db engine for information.
It is hard to trace codes with our eyes in a giant system like postgreSQL, so we can use GDB, a simple tool for debug. As my platform is on a mac, so we should install GDB with homebrew first:

	brew install gdb

Because of the security system of the new kernal for mac, GDB is not to allowed to control another process without permission of root. So we may make a signature for our GBD. This can be done in the **Keychain** function. So we just create a certificate as following step:

1. Create a new Certificate.
![]({{ site.baseurl }}/res/images/gdbpgsql/2.png)

2. Name the certificate and select type as **Code Signing**.
![]({{ site.baseurl }}/res/images/gdbpgsql/3.png)

3. Click continue until the specifying of location to store this certificate.
![]({{ site.baseurl }}/res/images/gdbpgsql/4.png)

4. Now the certificate has been established.
![]({{ site.baseurl }}/res/images/gdbpgsql/5.png)

5. Let's select **Always Trust** in `Get Info → Trust → Code Signing` when right-clicking.
![]({{ site.baseurl }}/res/images/gdbpgsql/6.png)

Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!