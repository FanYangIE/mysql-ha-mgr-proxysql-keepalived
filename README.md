# mysql-ha-mgr-proxysql-keepalived
This database system provides highly available, scalable, reliable and secure MySQL cluster services. This system can automatically distribute load and automatically transfer failures. As long as most nodes in the cluster are not down, it can continue to work normally and provide services to the outside world.

**tips:**

> Install the tool for uploading and downloading files in xshell:
> 
> ```yum -y install lrzsz```
> 
> Upload files with command: 'rz' (if overwriting the original file, use 'rz -y')
> 
> Download file: 'sz file_name'

### Configure MGR cluster

Install MySQL on all three servers
Configure /etc/hosts files on all three servers

```
10.2.18.24      VZDTHBI024
10.2.18.25      VZDTHMI025
10.2.18.27      VZWTDZI027
```
Also make sure the host name in the /etc/hostname file is consistent with the host name in /etc/hosts


### Operation on Node01 server

Modify the /etc/my.cnf configuration file of the node1 server

```
[mysqld]
basedir=/opt/mysql-5.7.36/
datadir=/opt/mysql-5.7.36/data
socket=/tmp/mysql.sock
symbolic-links = 0
character_set_server=utf8mb4
max_connections=2000
wait_timeout=600
interactive_timeout=600
lower_case_table_names = 1 #Table names are not case sensitive
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
log_timestamps=SYSTEM
innodb_buffer_pool_size=4G #Configure according to the specific memory size of the server
innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:10G #Specify the upper limit of the innodb temporary tablespace file size to prevent certain SQL statements from causing this file to explode and fill up the disk space.

#GTID:
server_id = 1
gtid_mode = on
enforce_gtid_consistency = on

master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE

#binlog
log_bin = mysql-bin
log-slave-updates = 1
binlog_format = row
expire_logs_days = 90

slave-skip-errors = 1007,1008,1032,1050,1051,1054,1060,1061,1062,1068,1091,1146,1217 #Ignore certain error codes to prevent cluster crashes.

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5db40c3c-180c-11e9-afbf-005056ac6820"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "10.2.18.23:33061"
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

> Parameter Description:
>
> If the Group Replication plugin has not been loaded when the server is started, the prefix used by the above variable loose-group_replication instructs the server to proceed with startup.
>
> Configuring transaction_write_set_extraction
>
> Instructs the server that for each transaction, it must collect the write set and encode it into a hash using the XXHASH64 hashing algorithm. Starting with MySQL 8.0.2, this setting is the default, so this line can be omitted.
>
> Configuring group_replication_group_name
>
> Must be a valid UUID. This UUID will be used internally when setting the GTID for Group Replication events in the binary log. Generate a UUID using SELECT UUID().
>
> Configuring group_replication_start_on_boot
>
> Instructs the plugin to not automatically start the operation on server startup. This is important when setting up Group Replication because it ensures that you can configure the server before manually starting the plugin.
>
> After configuring the members, you can set group_replication_start_on_boot to on to automatically start Group Replication when the server boots.
>
> Configuring group_replication_local_address
>
> Tells the plugin to use the network address 127.0.0.1 and port 24901 for internal communication with other members in the group.
>
> Important
>
> Group Replication uses this address for internal member-to-member connections using XCom. This address must be different from the hostname and port used for SQL and must not be used for client applications.
>
> You must reserve it for internal communication between group members while running Group Replication.
>
> The network address configured with group_replication_local_address
>
> must be resolvable by all group members. For example, if each server instance is on a different machine with a fixed network address, you can use the machine's IP, such as 10.0.0.1.
>
> If you use a host name, you must use a fully qualified name and ensure that it is resolvable via DNS, a properly configured /etc/hosts file, or other name resolution process.
>
> The recommended port for group_replication_local_address is 33061. In this tutorial, we use three server instances running on one machine, so ports 24901 to 24903 are used for internal communication network addresses.
>
> Configuring group_replication_group_seeds
>
> Set the hostname and port of the group member, which new members use to establish a connection to the group. These members are called seed members. Once a connection is established, group membership information is listed in performance_schema.replication_group_members.
>
> Typically, the group_replication_group_seeds list contains a list of hostname:port for each group member in group_replication_local_address, but this is not mandatory and a subset of group members can be selected as seeds.
>
> Important, the hostname:port column in group_replication_group_seeds is the internal network address of the seed member, as configured by group_replication_local_address ,
>
> and not the SQL hostname:port used for client connections and shown for example in the performance_schema.replication_group_members table.
>
> Add the following two lines for multi-primary mode: (remove the following two lines for single-primary mode)
>
> loose-group_replication_single_primary_mode=off
>
> loose-group_replication_enforce_update_everywhere_checks=on #This value must be consistent for group members, either all on or all off
>
> #IP address whitelist, only 127.0.0.1 is added by default, connections from external hosts are not allowed, security settings on demand
>
> loose-group_replication_ip_whitelist = '127.0.0.1/8,192.168.202.0/24'

reboot MySQL service

```
[root@VZDTHMI023 init.d]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

Log in to mysql to perform related settings

```
[root@VZDTHMI023 ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.36-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>SET SQL_LOG_BIN=0;
Query OK, 0 rows affected (0.00 sec)

mysql>GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%' IDENTIFIED BY 'yangfan';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.00 sec)

mysql>SET SQL_LOG_BIN=1;
Query OK, 0 rows affected (0.01 sec)

mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='yangfan' FOR CHANNEL 'group_replication_recovery';
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
| binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| sha256_password            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| MRG_MYISAM                 | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| CSV                        | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| InnoDB                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| INNODB_TRX                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCKS               | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCK_WAITS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_RESET           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM              | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM_RESET        | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX_RESET | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE_LRU     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_POOL_STATS   | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_TEMP_TABLE_INFO     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_METRICS             | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DEFAULT_STOPWORD | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DELETED          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_BEING_DELETED    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_CONFIG           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_CACHE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_TABLE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLES          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESTATS      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_INDEXES         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_COLUMNS         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FIELDS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN_COLS    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESPACES     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_DATAFILES       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_VIRTUAL         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| MyISAM                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| PERFORMANCE_SCHEMA         | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| MEMORY                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| FEDERATED                  | DISABLED | STORAGE ENGINE     | NULL                 | GPL     |
| BLACKHOLE                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| partition                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ARCHIVE                    | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ngram                      | ACTIVE   | FTPARSER           | NULL                 | GPL     |
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
45 rows in set (0.00 sec)

mysql> SET GLOBAL group_replication_bootstrap_group=ON;①
Query OK, 0 rows affected (0.00 sec)

mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (2.04 sec)

mysql> SET GLOBAL group_replication_bootstrap_group=OFF;②
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | ba65147e-5cb1-11ec-9e7e-005056864b71 | VZDTHMI023  |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
1 row in set (0.00 sec)
```

Make sure the status of the group_replication_applier above is "ONLINE"!

Note: Commands ① and ② are used to mark the servers that will be added to the cluster in the future as taking this server as the benchmark. Other servers do not need to be set. Used when setting up MGR for the first time or rebuilding MGR, it only needs to be enabled on one of the servers in the cluster.

### Operation on Node2 server

Modify the /etc/my.cnf configuration file of the node2 server

Compared with node1, the configuration file only has the asterisked items that are different.

```
[mysqld]
basedir=/opt/mysql-5.7.36/
datadir=/opt/mysql-5.7.36/data
socket=/tmp/mysql.sock
symbolic-links = 0
character_set_server=utf8mb4
max_connections=2000
wait_timeout=600
interactive_timeout=600
lower_case_table_names = 1 #Table names are not case sensitive
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
log_timestamps=SYSTEM
innodb_buffer_pool_size=4G #Configure according to the specific memory size of the server

#GTID:
server_id = 2 *
gtid_mode = on
enforce_gtid_consistency = on

master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE

#binlog
log_bin = mysql-bin
log-slave-updates = 1
binlog_format = row
expire_logs_days = 90

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5db40c3c-180c-11e9-afbf-005056ac6820"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "10.2.18.24:33061" *
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

reboot MySQL service

```
[root@VZDTHBI024 ~]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

### Log in to the node2 server to operate

```
[root@VZDTHBI024 ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.36-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SET SQL_LOG_BIN=0;
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%' IDENTIFIED BY 'yangfan';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql> SET SQL_LOG_BIN=1;
Query OK, 0 rows affected (0.00 sec)

mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='yangfan' FOR CHANNEL 'group_replication_recovery';
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
| binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| sha256_password            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| MRG_MYISAM                 | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| CSV                        | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| InnoDB                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| INNODB_TRX                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCKS               | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCK_WAITS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_RESET           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM              | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM_RESET        | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX_RESET | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE_LRU     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_POOL_STATS   | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_TEMP_TABLE_INFO     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_METRICS             | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DEFAULT_STOPWORD | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DELETED          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_BEING_DELETED    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_CONFIG           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_CACHE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_TABLE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLES          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESTATS      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_INDEXES         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_COLUMNS         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FIELDS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN_COLS    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESPACES     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_DATAFILES       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_VIRTUAL         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| MyISAM                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| PERFORMANCE_SCHEMA         | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| MEMORY                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| FEDERATED                  | DISABLED | STORAGE ENGINE     | NULL                 | GPL     |
| BLACKHOLE                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| partition                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ARCHIVE                    | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ngram                      | ACTIVE   | FTPARSER           | NULL                 | GPL     |
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
45 rows in set (0.00 sec)

mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (2.04 sec)

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 64c3a319-5cce-11ec-8651-005056868cff | VZDTHBI024  |        3306 | ONLINE       |
| group_replication_applier | e36bb851-5ccd-11ec-88bb-005056864b71 | VZDTHMI023  |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
2 rows in set (0.00 sec)
```

### Operations on Node3 server

Modify the /etc/my.cnf configuration file of the node3 server

Compared with node1, the configuration file only has the asterisked items that are different.


```
[mysqld]
basedir=/opt/mysql-5.7.36/
datadir=/opt/mysql-5.7.36/data
socket=/tmp/mysql.sock
symbolic-links = 0
character_set_server=utf8mb4
max_connections=2000
wait_timeout=600
interactive_timeout=600
lower_case_table_names = 1 #表名不区分大小写
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
log_timestamps=SYSTEM
innodb_buffer_pool_size=4G#根据实际情况确定

#GTID:
server_id = 3 *
gtid_mode = on
enforce_gtid_consistency = on

master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE

#binlog
log_bin = mysql-bin
log-slave-updates = 1
binlog_format = row
expire_logs_days = 90

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5db40c3c-180c-11e9-afbf-005056ac6820"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "10.2.18.25:33061" *
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

reboot MySQL service

```
[root@VZDTHBI024 ~]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

Log in to the node3 server to operate

```
[root@VZDTHBI024 ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.36-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> SET SQL_LOG_BIN=0;
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%' IDENTIFIED BY 'yangfan';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql> SET SQL_LOG_BIN=1;
Query OK, 0 rows affected (0.00 sec)

mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='yangfan' FOR CHANNEL 'group_replication_recovery';
Query OK, 0 rows affected, 2 warnings (0.01 sec)

mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
| binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| mysql_native_password      | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| sha256_password            | ACTIVE   | AUTHENTICATION     | NULL                 | GPL     |
| MRG_MYISAM                 | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| CSV                        | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| InnoDB                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| INNODB_TRX                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCKS               | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_LOCK_WAITS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP                 | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_RESET           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM              | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMPMEM_RESET        | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_CMP_PER_INDEX_RESET | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_PAGE_LRU     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_BUFFER_POOL_STATS   | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_TEMP_TABLE_INFO     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_METRICS             | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DEFAULT_STOPWORD | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_DELETED          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_BEING_DELETED    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_CONFIG           | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_CACHE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_FT_INDEX_TABLE      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLES          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESTATS      | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_INDEXES         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_COLUMNS         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FIELDS          | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_FOREIGN_COLS    | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_TABLESPACES     | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_DATAFILES       | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| INNODB_SYS_VIRTUAL         | ACTIVE   | INFORMATION SCHEMA | NULL                 | GPL     |
| MyISAM                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| PERFORMANCE_SCHEMA         | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| MEMORY                     | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| FEDERATED                  | DISABLED | STORAGE ENGINE     | NULL                 | GPL     |
| BLACKHOLE                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| partition                  | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ARCHIVE                    | ACTIVE   | STORAGE ENGINE     | NULL                 | GPL     |
| ngram                      | ACTIVE   | FTPARSER           | NULL                 | GPL     |
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+
45 rows in set (0.00 sec)

mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (2.04 sec)

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 64c3a319-5cce-11ec-8651-005056868cff | VZDTHBI024  |        3306 | ONLINE       |
| group_replication_applier | ba53f5a5-5cb1-11ec-a244-005056869823 | VZDTHMI025  |        3306 | ONLINE       |
| group_replication_applier | e36bb851-5ccd-11ec-88bb-005056864b71 | VZDTHMI023  |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
3 rows in set (0.00 sec)
```

At this point, the MGR cluster has been configured.

### Configure ProxySQL service

> 6032 is the management port number of ProxySQL;
> 6033 is the port number for external services.
> The username and password of ProxySQL are both the default admin

Modify the /etc/hosts file of all servers and proxysql servers in the MGR cluster

```
10.2.18.27  VZWTDZI027
10.2.18.24	VZDTHBI024
10.2.18.25	VZDTHMI025
10.2.18.22	VZDTHMI022
```

Install MySQL client on proxysql server
```
[root@VZDTHMI022 ~]# vim /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = https://yum.mariadb.org/10.6.5/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

[root@VZDTHMI022 ~]# yum install -y MariaDB-client
```

install proxysql

Download the latest installation package from GitHub https://github.com/sysown/proxysql/releases

https://github.com/sysown/proxysql/releases/download/v2.3.2/proxysql-2.3.2-1-centos7.x86_64.rpm

Planning read-write groups

|Default write group| Backup write group| Read group| Offline group|
|-|-|-|-|
|10 |20 |30 |40|

First install some dependency packages

```
yum install -y perl-DBI perl-DBD-MySQL libgnutls* gnutls
```

Install the proxysql package

```
[root@vzdthmi022 opt]# rpm -ivh proxysql-2.3.2-1-centos7.x86_64.rpm 
warning: proxysql-2.3.2-1-centos7.x86_64.rpm: Header V4 RSA/SHA256 Signature, key ID 79953b49: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:proxysql-2.3.2-1                 warning: group proxysql does not exist - using root
warning: group proxysql does not exist - using root
################################# [100%]
Created symlink from /etc/systemd/system/multi-user.target.wants/proxysql.service to /etc/systemd/system/proxysql.service.
```

Start the ProxySQL service

tips

> 6032 is the management port of ProxySQL. All configuration and management work of ProxySQL is logged in through this port;
> 6033 is the external service port of ProxySQL. The client accesses the backend database through this port.

```
[root@vzdthmi022 opt]# systemctl start proxysql
[root@vzdthmi022 opt]# ss -lntup|grep proxy
tcp    LISTEN     0      128       *:6032                  *:*                   users:(("proxysql",pid=3106,fd=44))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=3106,fd=35))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=3106,fd=34))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=3106,fd=32))
tcp    LISTEN     0      128       *:6033                  *:*                   users:(("proxysql",pid=3106,fd=31))

[root@VZDTHMI022 etc]# mysql -uadmin -padmin -h127.0.0.1 -P6032
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.5.30 (ProxySQL Admin Module)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.000 sec)
```
Explanation of proxysql database:

> • main library: It is the most important library of ProxySQL. It is the library used when the configuration needs to be modified. It is actually an in-memory database system. Therefore, after modifying the configuration in the main library, it must be persisted to disk to be permanently saved.
> • disk library: It is a disk database. The database structure is exactly the same as the in-memory database. When persisting the configuration in the in-memory database, it is actually written to the disk library. The default path of the disk database is $DATADIR/proxysql.db.
> • stats library: It is a statistical information library. The data in this library is generally temporarily filled when retrieving the data in it, and it is stored in memory. Because there is no related configuration item, it does not need to be persisted.
> • monitor library: It is a library related to monitoring the backend MySQL node. There are only a few log-type tables in this library. All the monitoring information collected by the monitoring module is stored in the corresponding log table.
> • stats_history library: It is a new library added in version 1.4.4, used to store historical statistical data. The default path is $DATADIR/proxysql_stats.db.

Initialize Proxysql and delete all previous proxysql data (if any)

```
MySQL [(none)]> delete from scheduler;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> delete from mysql_servers;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> delete from mysql_users;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> delete from mysql_query_rules;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> delete from mysql_group_replication_hostgroups;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.002 sec)

MySQL [(none)]> SAVE MYSQL VARIABLES TO DISK;
Query OK, 147 rows affected (0.004 sec)

MySQL [(none)]> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.003 sec)

MySQL [(none)]> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.014 sec)

MySQL [(none)]> LOAD MYSQL USERS TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE MYSQL USERS TO DISK;
Query OK, 0 rows affected (0.003 sec)

MySQL [(none)]> LOAD SCHEDULER TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE SCHEDULER TO DISK;
Query OK, 0 rows affected (0.006 sec)

MySQL [(none)]> LOAD MYSQL QUERY RULES TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE MYSQL QUERY RULES TO DISK;
Query OK, 0 rows affected (0.009 sec)
```

Create an account in the database cluster for proxysql login (operate on any of the three MGR nodes, and it will be automatically synchronized to other nodes) (If in single-master mode, execute on the writable MySQL node)

```
[root@VZDTHMI023 ~]# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 19
Server version: 5.7.36-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'proxysql'@'%' IDENTIFIED BY 'proxysql';
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL ON *.* TO  'proxysql'@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```

Create functions and views to check the status of MGR nodes in the database cluster
(Operate on any of the three MGR nodes, and it will be automatically synchronized to other nodes) (If in single-master mode, execute on a writable MySQL node)
On the master node of MGR, create the system view sys.gr_member_routing_candidate_status, which will provide group replication-related monitoring status indicators for ProxySQL.

Upload the addition_to_sys.sql script to the /opt directory of the writable member server of the MGR cluster, and execute the following statement to import it into MySQL (after executing it on mysql of one MGR node, it will be synchronized to the other two nodes).

**script for mysql 5.7**

```
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;
```

**script for mysql 8.0**

```
-- SET @TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
-- SET @@SESSION.SQL_LOG_BIN= 0;
-- SET @TEMP_READ_ONLY = @@GLOBAL.READ_ONLY;
-- SET @TEMP_SUPER_READ_ONLY = @@GLOBAL.SUPER_READ_ONLY;
-- SET @@GLOBAL.READ_ONLY = 0;
USE sys;
 
 
DROP VIEW IF EXISTS gr_member_routing_candidate_status;
 
DROP FUNCTION IF EXISTS IFZERO;
DROP FUNCTION IF EXISTS LOCATE2;
DROP FUNCTION IF EXISTS GTID_NORMALIZE;
DROP FUNCTION IF EXISTS GTID_COUNT;
DROP FUNCTION IF EXISTS gr_applier_queue_length;
DROP FUNCTION IF EXISTS gr_member_in_primary_partition;
DROP FUNCTION IF EXISTS gr_transactions_to_cert;
 
DELIMITER $$
 
CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$
 
CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$
 
CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$
 
CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$
 
CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$
 
CREATE FUNCTION gr_transactions_to_cert() RETURNS int(11)
    DETERMINISTIC
BEGIN
  RETURN (select  performance_schema.replication_group_member_stats.COUNT_TRANSACTIONS_IN_QUEUE AS transactions_to_cert
    FROM
        performance_schema.replication_group_member_stats where MEMBER_ID=@@SERVER_UUID );
END$$
 
CREATE FUNCTION my_server_uuid() RETURNS TEXT(36) DETERMINISTIC NO SQL RETURN (SELECT @@global.server_uuid as my_id);$$
 
CREATE VIEW gr_member_routing_candidate_status AS
    SELECT
        IFNULL((SELECT
                        IF(MEMBER_STATE = 'ONLINE'
                                    AND ((SELECT
                                        COUNT(*)
                                    FROM
                                        performance_schema.replication_group_members
                                    WHERE
                                        MEMBER_STATE != 'ONLINE') >= ((SELECT
                                        COUNT(*)
                                    FROM
                                        performance_schema.replication_group_members) / 2) = 0),
                                'YES',
                                'NO')
                    FROM
                        performance_schema.replication_group_members
                            JOIN
                        performance_schema.replication_group_member_stats rgms USING (member_id)
                    WHERE
                        rgms.MEMBER_ID = my_server_uuid()),
                'NO') AS viable_candidate,
        IF((SELECT
                    ((SELECT
                                GROUP_CONCAT(performance_schema.global_variables.VARIABLE_VALUE
                                        SEPARATOR ',')
                            FROM
                                performance_schema.global_variables
                            WHERE
                                (performance_schema.global_variables.VARIABLE_NAME IN ('read_only' , 'super_read_only'))) <> 'OFF,OFF')
                ),
            'YES',
            'NO') AS read_only,
        IFNULL(sys.gr_applier_queue_length(), 0) AS transactions_behind,
        IFNULL(sys.gr_transactions_to_cert(), 0) AS transactions_to_cert;$$
 
DELIMITER ;
-- SET @@SESSION.SQL_LOG_BIN = @TEMP_LOG_BIN;
-- SET @@GLOBAL.READ_ONLY = @TEMP_READ_ONLY;
-- SET @@GLOBAL.SUPER_READ_ONLY = @TEMP_SUPER_READ_ONLY;
```

Import the addition_to_sys.sql file data

```
[root@VZDTHMI023 opt]# mysql -uroot -p < /opt/addition_to_sys.sql
Enter password:
```

Now you can log in to the database to see this view.

```
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
```

Write the database nodes in the MGR cluster to the mysql_servers table of proxysql

Here, set hostgroup_id to 10

```
[root@VZDTHMI022 etc]# mysql -uadmin -padmin -h127.0.0.1 -P6032
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.5.30 (ProxySQL Admin Module)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> insert into mysql_servers (hostgroup_id, hostname, port) values(10,'10.2.18.24',3306);
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> insert into mysql_servers (hostgroup_id, hostname, port) values(10,'10.2.18.25',3306);
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> insert into mysql_servers (hostgroup_id, hostname, port) values(10,'10.2.18.27',3306);
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> select * from  mysql_servers;
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 10           | 10.2.18.25 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 10           | 10.2.18.24 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 10           | 10.2.18.27 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
3 rows in set (0.000 sec)

MySQL [(none)]> LOAD mysql servers TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE mysql servers TO DISK;
Query OK, 0 rows affected (0.020 sec)
```
**tips**

>Table mysql_servers: List of backend database cluster servers that ProxySQL connects to

Configuring the mysql_group_replication_hostgroups Table

```
insert into mysql_group_replication_hostgroups(writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind)  values(10,20,30,40,1,1,0,1000);

MySQL [(none)]> load mysql servers to runtime;
Query OK, 0 rows affected (0.004 sec)

MySQL [(none)]> save mysql servers to disk;
Query OK, 0 rows affected (0.020 sec)
```

**Configure read-write separation**

Let all SELECT operations be routed to the hostgroup with hostgroup_id 2. If not configured, all read and write operations are performed on the master node of the mgr cluster.

Check whether there are any read-write separation rules configured (newly configured proxies will not have them. If there are any previously configured rules, delete them first.)
```
MySQL [(none)]> select * from mysql_query_rules;
Empty set (0.000 sec)

# If there is a configuration, delete it
MySQL [(none)]> delete from mysql_query_rules;
MySQL [(none)]> commit;

```

```
MySQL [(none)]> insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(1,1,'^SELECT.*FOR UPDATE$',10,1);

MySQL [(none)]> insert into mysql_query_rules(rule_id,active,match_digest,destination_hostgroup,apply)values(2,1,'^SELECT',30,1);

MySQL [(none)]> LOAD MYSQL QUERY RULES TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE MYSQL QUERY RULES TO DISK;
Query OK, 0 rows affected (0.009 sec)

```

**View MGR configuration**

```
MySQL [(none)]> select * from mysql_group_replication_hostgroups\G
**************************** 1. row ***************************
writer_hostgroup: 10 # writer group
backup_writer_hostgroup: 20 # backup writer group
reader_hostgroup: 30 # reader group
offline_hostgroup: 40 # offline group
active: 1 # enabled or not
max_writers: 1 # maximum number of writer nodes
writer_is_also_reader: 0 # determines whether a node remains in the reader_hostgroup group to provide read services after being upgraded to a writer node (put into writer_hostgroup). If MGR multi-master mode needs to be set to 1
max_transactions_behind: 1000 # This field determines how many transactions of the writer node can be postponed at most
comment: NULL # comment
1 row in set (0.000 sec)
```

Add an account to access the mgr database cluster in proxysql

Table MySQL_users: List of users connected to ProxySQL and their credentials. Note that ProxySQL will also use the same credentials to connect to the backend server!

```
MySQL [(none)]> INSERT INTO MySQL_users(username,password,default_hostgroup) VALUES ('proxysql','proxysql',10);
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> UPDATE global_variables SET variable_value='monitor' where variable_name='mysql-monitor_username';
Query OK, 1 row affected (0.001 sec)

MySQL [(none)]> UPDATE global_variables SET variable_value='monitor' where variable_name='mysql-monitor_password';
Query OK, 1 row affected (0.001 sec)

MySQL [(none)]> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.004 sec)

MySQL [(none)]> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.018 sec)
```
To avoid the error "9006 - ProxySQL Error: connection is locked to hostgroup 1 but trying to reach hostgroup 2", set the following proxysql variables:

```
MySQL [(none)]> set mysql-set_query_lock_on_hostgroup=0; #set syntax compatibility
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.002 sec)

MySQL [(none)]> SAVE MYSQL VARIABLES TO DISK;
Query OK, 147 rows affected (0.004 sec)
```

> mysql-set_query_lock_on_hostgroup
> When activated (default since 2.0.6), multiplexing and query routing are disabled if a SET statement is used in a multi-statement command or if the SET statement parsing is unsuccessful. The client will remain bound to a single backend connection. Any SET statement that ProxySQL does not understand will disable multiplexing and routing.

Finally, you need to load the information of the global_variables, mysql_servers, mysql_users tables into RUNTIME, and further load it into DISK:

```
MySQL [(none)]> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.002 sec)

MySQL [(none)]> SAVE MYSQL VARIABLES TO DISK;
Query OK, 147 rows affected (0.004 sec)

MySQL [(none)]> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.004 sec)

MySQL [(none)]> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.023 sec)

MySQL [(none)]> LOAD MYSQL USERS TO RUNTIME;
Query OK, 0 rows affected (0.000 sec)

MySQL [(none)]> SAVE MYSQL USERS TO DISK;
Query OK, 0 rows affected (0.007 sec)

```

Add a monitoring user (monitor) in the MGR database cluster

>Note that users in mysql_users cannot be used as monitoring users.

```
create user monitor@'%' identified by 'monitor';
grant usage,replication client on *.* to monitor@'%';
grant select on sys.* to monitor;
flush privileges;
```

Verify the login to the backend mgr database cluster through proxysql

```
[root@VZDTHMI022 etc]# mysql -uproxysql -pproxysql -h 127.0.0.1 -P6033 -e"select @@hostname"
+------------+
| @@hostname |
+------------+
| VZDTHMI025 |
+------------+
```

Configure proxysql web interface monitoring (optional)

```
MySQL [(none)]> SET admin-web_enabled='true';

MySQL [(none)]> LOAD ADMIN VARIABLES TO RUNTIME;

MySQL [(none)]> SAVE ADMIN VARIABLES TO DISK ;

MySQL [(none)]> show variables like '%admin-web_enabled%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| admin-web_enabled | true  |
+-------------------+-------+
1 row in set (0.000 sec)

```

Access address: https://x.x.x.x:6080/

The account and password are both: stats

This page can only be viewed, and the running configuration of proxysql cannot be modified.

**Configure proxysql cluster**

Add another proxysql node based on the single proxysql node

Set the cluster management account (executed on all proxysql nodes)
```
update global_variables set variable_value='admin:admin;cluster:cluster' where variable_name='admin-admin_credentials';
update global_variables set variable_value='cluster' where variable_name='admin-cluster_username';
update global_variables set variable_value='cluster' where variable_name='admin-cluster_password';

```

Set up ProxySQL cluster members (executed on all ProxySQL server nodes)

```
insert into proxysql_servers(hostname,port,weight,comment) values('10.2.18.22',6032,1,'primary'),('10.2.18.30',6032,1,'secondary');
```

**Load configuration**

First load the first ProxySQL node with existing backend server configuration (already configured proxysql node), then load the newly installed node.

```
load admin variables to runtime;
load proxysql servers to runtime;
save admin variables to disk;
save proxysql servers to disk;
```

**View the proxysql cluster status**

```
select hostname,port,comment,Uptime_s,last_check_ms from stats_proxysql_servers_metrics;
select hostname,name,checksum,updated_at from stats_proxysql_servers_checksums;
```

At this point, the newly added proxysql node has synchronized the configuration data from the original proxysql, such as the backend server (mysql_servers), users (mysql_users), etc.

**Configure automatic failover (keepalived) of the proxysql cluster**

Install the keepalived software on both proxysql nodes

```
yum install -y keepalived
```

Back up the original configuration file after installation

```
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
```

Create a script file to detect the proxysql service status

```
touch /etc/keepalived/check_proxysql.sh

File contents:
#!/bin/bash
/usr/bin/netstat -na | grep -e '0.0.0.0:6033' -e '0.0.0.0:6032' &>/dev/null

```

Install the net-tools package

```
yum install net-tools
```

Modify the keepalived service configuration file
vi /etc/keepalived/keepalived.conf
Because the configured proxysql1 has a higher priority than proxysql2 in the keepalived service, after proxysql1 fails and recovers, it will be forced to take over the service again.

**Configuration file content (proxysql1)**

```
! Configuration File for keepalived

global_defs {
   router_id proxysql_ha
}

#Define a script to check whether the proxysql service is alive
vrrp_script chk_proxysql_port {
#Call the proxysql script file
script "/etc/keepalived/check_proxysql.sh"

#Script execution interval, check once every 1s
interval 1

#Priority change caused by script results, if the test fails (the script returns non-0), the priority will be reduced by 50
weight -50

#The service is truly unavailable only after two consecutive failures. The priority will be reduced by weight (between 1-255)
fall 2

#The test is considered successful if it succeeds twice. But the priority is not changed
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    mcast_src_ip 10.2.18.22
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
      10.2.18.36
    }

    track_script {
      chk_proxysql_port
    }
}

```

**配置文件内容（proxysql2）**

```
! Configuration File for keepalived

global_defs {
   router_id proxysql_ha
}

#Define a script to check whether the proxysql service is alive
vrrp_script chk_proxysql_port {
#Called proxysql script file
script "/etc/keepalived/check_proxysql.sh"

#Script execution interval, check once every 1s
interval 1

#Priority change caused by script results, if the test fails (the script returns non-0), the priority is reduced by 50
weight -50

#The test is considered to be a real failure only if it fails twice in a row. The priority will be reduced by weight (between 1-255)
fall 2

#The test is considered successful if it succeeds twice. But the priority is not changed
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    mcast_src_ip 10.2.18.30
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
      10.2.18.36
    }

    track_script {
      chk_proxysql_port
    }
}

```

**tips：**

If two nodes are mounted to the virtual IP at the same time, add the following firewall rules:
```
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface ens192 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --reload
```
