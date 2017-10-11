---
layout: post
title: Recover from GTID replication relay log error in MySQL 5.6 using GTID_PURGED
---

Recently, we had an error in our production system. Replication on MySQL Slave failed.
It was not caused by a bug, but by an unexpected Slave server restart of the whole machine.
After checking ```SHOW SLAVE STATUS\G``` we got scared, that Master's binlog is also corrupt,
which happened not so long ago - [MySQL Bug #7022](http://bugs.mysql.com/bug.php?id=70222).

Fortunately, Master's binlog was intact, so we moved back to Slave, to solve the problem (removed MySQL noise for brevity):

~~~
mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
...
Slave_IO_Running: Yes
Slave_SQL_Running: No
...
Last_Errno: 1594
Last_Error: Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master's binary log is corrupted (you can check this by running 'mysqlbinlog' on the binary log), the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' on the relay log), a network problem, or a bug in the master's or slave's MySQL code. If you want to check the master's binary log or slave's relay log, you will be able to know their names by issuing 'SHOW SLAVE
...
Executed_Gtid_Set: 3aabcfda-591d-11e3-81f1-0050569a4585:1-14351185
Auto_Position: 1
1 row in set (0.00 sec)
~~~

You can see from the output, that we are using [GTID replication](http://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html),
which is available since MySQL 5.6. Transactions ```1-14351185``` were executed successfully, but the ```14351186``` got corrupt.
We tried to skip it by
[committing an empty GTID transaction](http://www.mysqlperformanceblog.com/2013/03/26/repair-mysql-5-6-gtid-replication-by-injecting-empty-transactions/),
but that did not help.

What to do next? At this point I wanted to start crying, because multiple sites and tutorials suggest to
perform a clean ```mysqldump``` of Master, which will be copied to Slave.
That is _"the correct"_ way of solving replication problems.
However, shutting down a site, just to perform a _~700GB database dump_ is not fun at all.



## MySQL 5.6 replication variables

Browsing through the net reveals a small number of pages, which mention this secret ```GTID_PURGED``` variable.

* [http://www.mysqlperformanceblog.com/2013/02/08/how-to-createrestore-a-slave-using-gtid-replication-in-mysql-5-6/](http://www.mysqlperformanceblog.com/2013/02/08/how-to-createrestore-a-slave-using-gtid-replication-in-mysql-5-6/)
* [http://dev.mysql.com/doc/refman/5.6/en/replication-options-gtids.html](http://dev.mysql.com/doc/refman/5.6/en/replication-options-gtids.html)

I didn't comprehend the meaning of this variable at first (straight from dev.mysql.com):

> The set of all transactions that have been purged from the binary log.

But after some digging, it has definitely its place inside MySQL.

By changing the ```GTID_PURGED``` on Slave, we can **inform the server about missing binary logs on Master, so Slave
starts replication from the next available transaction identifier**. This way, we don't have to perform a full
Master <i class="fa fa-long-arrow-right"></i> Slave backup. Just make sure, that binlogs, which you want to synchronize
are still present on Master.



## Work your magic GTID_PURGED!

1.
	**Reset available replication information on Slave**, so Slave forgets its own binlogs and Master's relay logs.

~~~
mysql> RESET SLAVE;
mysql> RESET MASTER;
~~~

2.
	Issue a command to **reconfigure replication** by pointing Slave to Master (which was cleared in previous step):

~~~
mysql> CHANGE MASTER TO
MASTER_HOST = '...',
MASTER_PORT = ...,
MASTER_USER = '...',
MASTER_PASSWORD = '...',
MASTER_AUTO_POSITION = 1;
~~~

3.
	We are ready to instruct Slave with information about _deleted binlogs on Master_, so Slave will not replicate them and
	they will be skipped.

	_The reason, why we want to skip some transactions, is because they were already replicated to Slave before._

~~~
mysql> SET global GTID_PURGED="3aabcfda-591d-11e3-81f1-0050569a4585:1-14351185";
~~~

4.
	We are done with preparations. All, that is left to be done is to **start the replication**:

~~~
mysql> START SLAVE;
~~~

5.
	And check if both Slave replication threads are doing their work without errors:

~~~
*************************** 1. row ***************************
Slave_IO_State: Waiting for master to send event
...
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
...
Seconds_Behind_Master: 157717
Last_IO_Errno: 0
Last_IO_Error:
Last_SQL_Errno: 0
Last_SQL_Error:
...
Retrieved_Gtid_Set: 3aabcfda-591d-11e3-81f1-0050569a4585:14351185-14805026
Executed_Gtid_Set: 3aabcfda-591d-11e3-81f1-0050569a4585:1-14351185
Auto_Position: 1
1 row in set (0.00 sec)
~~~



MySQL is a great database, which solves problems it is meant to solve. New replication functionality
(GTID) however has it quirks when compared to the "old"
[replication by master log coordinates](http://dev.mysql.com/doc/refman/5.5/en/replication-howto-masterstatus.html).
Hopefully, this post will help others when solving similar problems with **bringing replication back to life after Slave failure**.
