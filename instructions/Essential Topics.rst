Essential Topics
=========================================

This module will cover topics crucial to working with PXC.  This section assumes you have completed the 'Migrate Master Slave to Cluster' section.

.. contents:: 
   :backlinks: entry
   :local:


Incremental State Transfers
-----------------------------

We've got a barebones configuration on all 3 of our cluster nodes, and each node (except the first) was synchronized to the cluster via SST -- a full backup of another node.  Luckily, this is not required every time you restart a node, but let's test it out and see how it works.  

What is IST?
~~~~~~~~~~~~~~~~~~~~~~~~~

IST is used to avoid full SST for temporary node outages.  The general idea is that if a node goes down in a known state, misses some replication events, and them comes back online, the cluster should be able to feed those events to that node without requiring a full State Transfer.  

Each node has something called the Gcache, which is temporary local storage of writesets that have been replicated across the cluster.  Provided all the needed replication events can be found in some working node in the cluster, IST can be performed.  

Note that IST uses a completely separate port from SST.  

Checking IST configuration
~~~~~~~~~~~~~~~~~~~~~~~~~

The IST configuration is buried in the ``wsrep_provider_options`` MySQL variable::

	node3 mysql> show variables like 'wsrep_provider_options'\G
	*************************** 1. row ***************************
	Variable_name: wsrep_provider_options
	        Value: base_host = 192.168.70.4; base_port = 4567; evs.debug_log_mask = 0x1; \
		evs.inactive_check_period = PT0.5S; evs.inactive_timeout = PT15S; evs.info_log_mask = 0; \
		evs.install_timeout = PT15S; evs.join_retrans_period = PT0.3S; evs.keepalive_period = PT1S; \ 
		evs.max_install_timeouts = 1; evs.send_window = 4; evs.stats_report_period = PT1M; \
		evs.suspect_timeout = PT5S; evs.use_aggregate = true; evs.user_send_window = 2; evs.version = 0; \
		evs.view_forget_timeout = PT5M; gcache.dir = /var/lib/mysql/; gcache.keep_pages_size = 0; \
		gcache.mem_size = 0; gcache.name = /var/lib/mysql//galera.cache; gcache.page_size = 128M; \
		gcache.size = 128M; gcs.fc_debug = 0; gcs.fc_factor = 0.5; gcs.fc_limit = 16; gcs.fc_master_slave = NO; \
		gcs.max_packet_size = 64500; gcs.max_throttle = 0.25; gcs.recv_q_hard_limit = 9223372036854775807; \
		gcs.recv_q_soft_limit = 0.25; gcs.sync_donor = NO; gmcast.listen_addr = tcp://0.0.0.0:4567; \ 
		gmcast.mcast_addr = ; gmcast.mcast_ttl = 1; gmcast.peer_timeout = PT3S; gmcast.time_wait = PT5S; \ 
		gmcast.version = 0; ist.recv_addr = 192.168.70.4; pc.checksum = true; pc.ignore_quorum = false; \ 
		pc.ignore_sb = false; pc.linger = PT2S; pc.npvo = false; pc.version = 0; protonet.backend = asio; \ 
		protonet.version = 0; replicator.causal_read_timeout = PT30S; replicator.commit_order = 3
	1 row in set (0.00 sec)

Specifically, look for ``ist.recv_addr``.


Testing IST
~~~~~~~~~~~~~~~~~~~~~~~~~

Startup sysbench again on node1 (you may still have this running)::

	[root@node1 ~]# sysbench --test=sysbench_tests/db/oltp.lua \
		--mysql-host=node1 --mysql-user=test --mysql-db=test \
		--oltp-table-size=250000 --report-interval=1 --max-requests=0 \
		--tx-rate=10 run | grep tps

Now, we have traffic writing to node1.  

**Startup application traffic on node1**

Go to node3.  Watch the error log, and in another window restart mysql::

	[root@node3 ~]# tail -f /var/lib/mysql/node3.err 
	[root@node3 ~]# service mysql stop

Also watch ``myq_status`` on the other nodes to see how the cluster behaves.

**Restart node3's mysql and watch the error log**

- What happened to the cluster soon as you issued the restart?
- Did node3 restart correctly?
- Did it rejoin without SST?

Scan the error.log on node3 for references to IST.  You should see (amongst a lot of output)::

	120810 19:07:40 [Warning] WSREP: Gap in state sequence. Need state transfer.
	...
	120810 19:07:42 [Note] WSREP: Prepared IST receiver, listening at: tcp://192.168.70.4:4568
	...
	120810 19:07:43 [Note] WSREP: Receiving IST: 14 writesets, seqnos 1349-1363
	...
	120810 19:07:43 [Note] WSREP: 1 (node2): State transfer to 0 (node3) complete.

- What port did IST use on node3?
- What are the limitations of IST?


What happens if IST doesn't work
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Unfortunately it's a bit confusing to setup IST sometimes, and it can get frustrating.  The ``wsrep_node_address`` option makes things a little easier, but before it came along, you had to configure ``ist.recv_addr`` directly in the ``wsrep_provider_options``.  Additionally, in setups with both public and private networks that you want to use for different things, you will have to still set this directly.  

Currently ``ist.recv_addr`` is set to our node's ip.  Let's simulate it being misconfigured by adding this line to our my.cnf on node3::

	[mysqld]
	...
	wsrep_provider_options="ist.recv_addr="

Now restart node3 again and make observations.

**Misconfigure node3's IST and restart again**

- What happens?
- How many times do you need to restart MySQL?
- What happens before it rejoins the cluster?

Failure to start mysql can trigger a node to require a full SST on the next start.  It's important to be careful about this.

**Remove node3's misconfiguration and restart it again**



Rolling cluster restarts
----------------------------------------

Let's spruce up the configuration a bit and take the opportunity to do a rolling restart on the cluster.  

Our cluster configuration should look something like this now on node1::

	[mysqld]
	datadir=/var/lib/mysql

	server-id=1
	binlog_format=ROW
	log-slave-updates

	# galera settings
	wsrep_provider                  = /usr/lib/libgalera_smm.so

	wsrep_cluster_name              = mycluster
	wsrep_cluster_address           = gcomm://192.168.70.2,192.168.70.3,192.168.70.4
	wsrep_node_name                 = node1
	wsrep_node_address              = 192.168.70.2

	wsrep_sst_method                = xtrabackup
	wsrep_sst_auth                  = sst:secret

	# innodb settings for galera
	innodb_locks_unsafe_for_binlog   =  1
	innodb_autoinc_lock_mode         =  2


Let's change the configuration to something like this::

	[mysqld]
	datadir                         = /var/lib/mysql
	log_error                       = error.log
	
	binlog_format                   = ROW
	
	innodb_log_file_size            = 64M
	
	wsrep_cluster_name              = mycluster
	wsrep_cluster_address           = gcomm://192.168.70.2,192.168.70.3,192.168.70.4
	wsrep_node_name                 = node1
	wsrep_node_address              = 192.168.70.2
	
	wsrep_provider                  = /usr/lib/libgalera_smm.so
	
	wsrep_sst_method                = xtrabackup
	wsrep_sst_auth                  = sst:secret
	
	wsrep_slave_threads             = 2
	
	innodb_locks_unsafe_for_binlog  = 1
	innodb_autoinc_lock_mode        = 2
	
	[mysql]
	prompt                          = "node1 mysql> "
	
	[client]
	user                            = root

These are all minor changes, but will make it easier to do more exercises in the tutorial.  

Let's start with node1.  Shutdown mysql there and pay close attention to ``myq_status`` on the other nodes when you do it::

	[root@node1 ~]# service mysql stop

Now edit your my.cnf with the above configuration.

**On node1, shutdown mysql, make your configuration changes, and restart MySQL**

- Did MySQL restart?  If not, check the logs to find out why. (note that we renamed the error log)

Turns out we increased the default innodb_log_file_size, and we forgot to remove the old logs::

	[root@node1 ~]# rm -f /var/lib/mysql/ib_iblog*

**Remove the innodb logs on node1 and try again**

- Does the cluster restart this time?

Now we should be ready to make our config changes on nodes 2 and 3.  Follow the same procedure, but be careful to modify the configuration elements that need to be unique on each host before restarting the node.

**Apply the config changes to each of the other two nodes in turn**

This technique can be used to make non-dynamic cluster configuration changes, and to upgrade PXC to the latest release.  Just do one node at a time.  

- Did any of your nodes SST?



Application Interaction with the Cluster
----------------------------------------

Reading and Writing to the Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is recommended that you run ``myq_status -t 1 wsrep`` on each node in a terminal window (or windows) that you can easy glance at.  This will show you the status of the cluster at a glance.  

Pick a node (any node) and create a new table in the ``test`` schema::

	node2 mysql> create table test.autoinc ( i int unsigned not null auto_increment primary key, j varchar(32) );
	Query OK, 0 rows affected (0.02 sec)

	node2 mysql> show create table test.autoinc\G
	*************************** 1. row ***************************
	       Table: autoinc
	Create Table: CREATE TABLE `autoinc` (
	  `i` int(10) unsigned NOT NULL AUTO_INCREMENT,
	  `j` varchar(32) DEFAULT NULL,
	  PRIMARY KEY (`i`)

Now, let's insert some data into this table::

	node2 mysql> insert into test.autoinc (j) values ('node2' );
	Query OK, 1 row affected (0.00 sec)

	node2 mysql> insert into test.autoinc (j) values ('node2' );
	Query OK, 1 row affected (0.01 sec)

	node2 mysql> insert into test.autoinc (j) values ('node2' );
	Query OK, 1 row affected (0.00 sec)

Now select all the data in the table::

	node2 mysql> select * from test.autoinc

**Create the test.autoinc table on node2, insert some rows, and check the data**
	
- Does anything strike you as odd about the rows?
- What happens if you do the inserts on each node in order?

**Experiment with the inserts and pay special attention to the autoincrement column**

- How would the default autoincrement behavior affect your application?

*Information about how to modify this behavior is later in the tutorial* 


Deadlocks when you don't expect them
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

One of the things to be aware of with using PXC is that there can be rollbacks issued by the server on COMMIT (and other unexpected parts of transactions), which cannot happen in standard single-node Innodb.

To illustrate this, open a mysql session on two nodes and follow these steps carefully::

	node1 mysql> set autocommit=off;
	node1 mysql> select * from test.autoinc;
	node1 mysql> update test.autoinc set j="node1" where i = 1;

**NOTE**: you may need to select another row.  Just be sure you always select a row that exists and has a value that your UPDATE will actually *change*.

We now have an open transaction on node1 with a lock on a single row.  If we run ``SHOW ENGINE INNODB STATUS\G``, we can see the transaction open with a record lock::

	------------
	TRANSACTIONS
	------------
	...
	---TRANSACTION 83B, ACTIVE 50 sec
	2 lock struct(s), heap size 376, 1 row lock(s), undo log entries 1
	MySQL thread id 3972, OS thread handle 0x7fddb84e0700, query id 16408 localhost root sleeping
	show engine innodb status
	Trx read view will not see trx with id >= 83C, sees < 83C
	TABLE LOCK table `test`.`autoinc` trx id 83B lock mode IX
	RECORD LOCKS space id 0 page no 823 n bits 72 index `PRIMARY` of table `test`.`autoinc` trx id 83B lock_mode X locks rec but not gap

**Open a non-autocommit transaction on node1 and update a row in the test.autoinc table, but do not commit**


While the transaction is still open, go try to modify the row on another node::

	node3 mysql> set autocommit=off;
	node3 mysql> select * from test.autoinc;
	node3 mysql> update test.autoinc set j="node3" where i=1;
	node3 mysql> commit;

**Do the same thing on node3, but set j to a different value, and commit**

- Does the transaction succeed?
- What is the value of that record on node3?  Was it set correctly?

This commit succeeded!  On standard Innodb, this should have blocked waiting for the row lock to be released by the first transaction.  Let's go back and see what happens if we try to commit on node1::

	node1 mysql> commit;

**Commit on node1**

- What happens on node1?
- What is the value of j in the row on node1?

We get a deadlock on node1, in spite of it being the first transaction to open a record lock.  

- Retry these steps, but instead of a ``commit`` on node1, do another select.  What is the result?
- Retry these steps, but instead of two separate nodes, execute them in different sessions on the same node.  What is the result?
- Imagine this is your production environment and you are seeing these deadlocks.  How would you troubleshoot this?
- Does the deadlock show up in ``SHOW ENGINE INNODB STATUS``?

**Experiment further with this behavior until you understand it**


Application hotspots
~~~~~~~~~~~~~~~~~~~~~~

The more your application updates a small subset of your data, the more likely the above conflicts will happen.  

First, let's setup a test so we can see when these deadlocks happen.  There is monitoring available in ``myq_status``` that you can use to see when these deadlocks occur.  They are in the 'Conflct' column, entitled 'lcf' and 'bfa' for "Local Certification failures" and "Brute force aborts", respectively.  For a full description of what those mean, read `this blog post. <http://www.mysqlperformanceblog.com/2012/11/26/realtime-stats-to-pay-attention-to-in-percona-xtradb-cluster-and-galera>`_.

Startup ``myq_status`` on two of your nodes and check those columns.  On the same two nodes, startup sysbench::

	sysbench --test=sysbench_tests/db/oltp.lua \
	--mysql-user=test --mysql-db=test \
	--oltp-table-size=250000 --report-interval=1 --max-requests=0 \
	--tx-rate=10 run | grep tps

*note that I removed the --mysql-host option -- this defaults to the local server**

**Start sysbench on two nodes and monitor with myq_status**

- How many lcf's and bfa's do you see?

It is most likely the case that you don't see any.  This sysbench is doing writes spread out across all 250k rows in a single table.  As these conflicts happen more readily with a smaller working set, simply re-start sysbench with a smaller ``--oltp-table-size``::

	sysbench --test=sysbench_tests/db/oltp.lua \
	--mysql-user=test --mysql-db=test \
	--oltp-table-size=25000 --report-interval=1 --max-requests=0 \
	--tx-rate=10 run | grep tps

*Note: there should be no need to do a sysbench cleanup and prepare*

**Keep decreasing the table size in sysbench until you see some bfas**

- What working set did you need to reduce the test to until you started seeing brute force aborts?
- What is more common?  BFA or LCF?

If you check the sysbench command line more closely, you'll see a ``--tx-rate`` option.  This is limiting the speed of the sysbench test to 10 transactions per second.  Let's remove that and see how it affects the conflict rate.  Note that this will increase the CPU utilization on your system, so you probably don't want to leave it running very long::

	sysbench --test=sysbench_tests/db/oltp.lua \
	--mysql-user=test --mysql-db=test \
	--oltp-table-size=2500 --report-interval=1 --max-requests=0 \
	run | grep tps

At this point you should be getting bfas regularly.  Keep reducing the table size until you see some lcfs. It may take a while to see an lcf.

You may also need to do the following to get the conditions right to see an lcf:

- set global innodb_flush_log_at_trx_commit=0
- set global wsrep_slave_threads=1

**Remove the tx-rate option and keep reducing the working set until you see at least one lcf**

- What does it take to get an lcf?
- Why are lcfs so much less common than bfas (at least in this environment)?

**Do a rolling restart once you are done with this exercise to reset your settings back to the defaults**


Monitoring Galera
-------------------

We've already been using ``myq_status`` to check Galera status.  It pulls data from::

	mysql> SHOW GLOBAL VARIABLES LIKE 'wsrep%';
	mysql> SHOW GLOBAL STATUS LIKE 'wsrep%';

Feel free to use the documentation for these `settings<http://www.codership.com/wiki/doku.php?id=mysql_options_0.8>`_ and `status variables<http://www.codership.com/wiki/doku.php?id=galera_status_0.8>`_.


**Run those commands on a node (or nodes) in your cluster and try to see how they line up with myq_status**


Online Schema Changes
-----------------------

It's important to know how to make schema changes within the cluster.  Restart the original sysbench on node1::

	[root@node1 ~]# sysbench --test=sysbench_tests/db/oltp.lua \
		--mysql-host=node1 --mysql-user=test --mysql-db=test \
		--oltp-table-size=250000 --report-interval=1 --max-requests=0 \
		--tx-rate=10 run | grep tps

**Restart a rate limited sysbench on node1**

Basic Alter Table
~~~~~~~~~~~~~~~~~~~~

Now that we have a basic workload running, let's see the effect of altering that table. Note that we will be using the `Total Order Isolation <http://www.codership.com/wiki/doku.php?id=rolling_schema_upgrade#total_order_isolation_toi>`_ default setting for Galera.  

In another window, run an ALTER TABLE on test.sbtest1::

	node1 mysql> alter table test.sbtest1 add column `m` varchar(32);

**Add an extra column to test.sbtest1 on node1**

- How is our application affected by this change? (i.e., What happens to the tps that sysbench is reporting?)


Now, let's go to node2 and do a similar operation::

	node2 mysql> alter table test.sbtest1 add column `n` varchar(32);

**Add an extra column to test.sbtest1 on node2**

- Does running the command from another node make a difference?


Create a copy of our test table with data from the original table::

	node2 mysql> create table test.foo like test.sbtest1;	
	node2 mysql> insert into test.foo select * from test.sbtest1;

**Create and populate test.foo using the above commands**

- What effect does this have on the cluster?

Now let's run the ALTER on this new table::

	node2 mysql> alter table test.foo add column `o` varchar(32);

**Alter test.foo**

- How long does the alter take?
- How long is sysbench blocked for?  Why?


Rolling Schema Upgrades
~~~~~~~~~~~~~~~~~~~~~~~~~~

Galera provides a `Rolling Schema Upgrade  <http://www.codership.com/wiki/doku.php?id=rolling_schema_upgrade>`_ setting to allow you to avoid globally locking the cluster on a schema change.  Let's try it out, set this global variable on all three nodes::

	node1 mysql> set global wsrep_OSU_method='RSU';
	node2 mysql> set global wsrep_OSU_method='RSU';
	node3 mysql> set global wsrep_OSU_method='RSU';


Add another column to ``test.foo`` (the table that sysbench is *not* modifying)::

	node2 mysql> alter table test.foo add column `p` varchar(32);

**Set wsrep_OSU_method to RSU and add another column to test.foo on node2**

- What is the effect on our live workload?


Add a column on ``test.sbtest``, but on node2::

	node2 mysql> alter table test.sbtest1 add column `q` varchar(32);

**Add another column to test.sbtest1 on node2**

- What is the effect on our live workload?
- How do we need to propagate this change around our cluster?  How can we do it without stopping our application?


Finally, let's drop a column on ``test.sbtest1`` that sysbench is using (you may want to watch myq_status while you do this)::

	node2 mysql> alter table test.sbtest1 drop column `c`;

**Drop column `c` from test.sbtest1 on node2**

- What happened to node2?
- How did it affect the application workload?
- What is the limitation of using the Rolling Schema Upgrade feature?

**Restart mysql on node2 to trigger SST**


pt-online-schema-change
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is not a tutorial on `pt-online-schema-change <http://www.percona.com/doc/percona-toolkit/pt-online-schema-change.html>`_, but let's illustrate that it works with PXC.

First, set the ``wsrep_OSU_method`` back to TOI (the default) on all nodes::

	node1 mysql> set global wsrep_OSU_method='TOI';
	node2 mysql> set global wsrep_OSU_method='TOI';
	node3 mysql> set global wsrep_OSU_method='TOI';

Now, let's do our schema change fully non-blocking::

	[root@node2 ~]# pt-online-schema-change --alter "add column z varchar(32)" D=test,t=sbtest1 --execute

**Run pt-osc from node2**

- Does this work?  If not, why? (hint: check for Conflicts)
- What can make it work in this case? (`hint<http://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--retries>`_)

**Make necessary adjustments to get the pt-osc completed**
