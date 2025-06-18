# mysql-ha-mgr-proxysql-keepalived
This database system provides highly available, scalable, reliable and secure MySQL cluster services. This system can automatically distribute load and automatically transfer failures. As long as most nodes in the cluster are not down, it can continue to work normally and provide services to the outside world.

tips:
xshell中安装上传和下载文件的工具：
yum -y install lrzsz

上传文件用：“rz”（如果覆盖原文件用“rz -y”）
下载文件用：“sz 文件名”

### 配置MGR集群

3台服务器全部安装好MySQL
在3台服务器里全部配置/etc/hosts文件

```
10.2.18.24      VZDTHBI024
10.2.18.25      VZDTHMI025
10.2.18.22      VZDTHMI022
10.2.18.27      VZWTDZI027
```
同时确保/etc/hostname文件中的主机名和/etc/hosts中的主机名保持一致


Node01服务器上操作
修改node1服务器的/etc/my.cnf配置文件

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
innodb_temp_data_file_path = ibtmp1:12M:autoextend:max:10G #指定innodb临时表空间文件上限值大小，防止某些SQL导致此文件暴涨，挤爆磁盘空间。

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

slave-skip-errors = 1007,1008,1032,1050,1051,1054,1060,1061,1062,1068,1091,1146,1217 #忽略指定错误，防止导致集群崩溃。

transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5db40c3c-180c-11e9-afbf-005056ac6820"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address= "10.2.18.23:33061"
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

> 参数说明：
> 如果在启动服务器时尚未加载组复制插件，则上述变量loose-使用 的前缀 group_replication指示服务器继续启动。
> 
> 配置 transaction_write_set_extraction
> 
> 指示服务器对于每个事务，它必须收集写集并使用XXHASH64散列算法将其编码为散列。从MySQL 8.0.2开始，此设置是默认设置，因此可以省略此行。
> 
> 配置 group_replication_group_name
> 
> 必须是有效的UUID。在二进制日志中为组复制事件设置GTID时，将在内部使用此UUID。使用SELECT UUID()生成一个UUID。
> 
> 配置 group_replication_start_on_boot
> 
> 指示插件在服务器启动时不自动启动操作。这在设置组复制时很重要，因为它确保您可以在手动启动插件之前配置服务器。
> 
> 配置成员后，可以设置 group_replication_start_on_boot 为on，以便在服务器引导时自动启动Group Replication。
> 
> 配置 group_replication_local_address
> 
> 告诉插件本机使用网络地址127.0.0.1和端口24901与组中的其他成员进行内部通信。
> 
> 重要
> 
> 组复制使用此地址进行使用XCom的内部成员到成员连接。此地址必须与用于SQL的主机名和端口不同，并且不得用于客户端应用程序。
> 
> 在运行组复制时，必须为组成员之间的内部通信保留它。
> 
> 配置的网络地址 group_replication_local_address
> 
> 必须可由所有组成员解析。例如，如果每个服务器实例位于具有固定网络地址的其他计算机上，则可以使用计算机的IP，例如10.0.0.1。
> 
> 如果使用主机名，则必须使用完全限定名称，并确保它可通过DNS，正确配置的/etc/hosts 文件或其他名称解析过程进行解析。
> 
> 建议的端口 group_replication_local_address 是33061.在本教程中，我们使用在一台计算机上运行的三个服务器实例，因此端口24901到24903用于内部通信网络地址。
> 
> 配置 group_replication_group_seeds
> 
> 设置组成员的主机名和端口，新成员使用它们建立与组的连接。这些成员称为种子成员。建立连接后，将列出组成员身份信息 performance_schema.replication_group_members。
> 
> 通常，group_replication_group_seeds 列表包含hostname:port每个组成员的列表 group_replication_local_address，但这不是强制性的，可以选择组成员的子集作为种子。
> 
> 重要，该hostname:port列在 group_replication_group_seeds 是种子构件的内部网络地址，由被配置 group_replication_local_address ，
> 
> 而不是SQL hostname:port用于客户端连接，并且例如在显示 performance_schema.replication_group_members 表中。
> 
> 多主模式添加以下两行：（单主模式则去掉下面两行配置）
> loose-group_replication_single_primary_mode=off     
> loose-group_replication_enforce_update_everywhere_checks=on  #组成员此值必须统一，要么全为on要么全为off
> 
> #IP地址白名单,默认只添加127.0.0.1,不会允许来自外部主机的连接,按需安全设置
> loose-group_replication_ip_whitelist = '127.0.0.1/8,192.168.202.0/24'

重启MySQL服务

```
[root@VZDTHMI023 init.d]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

登录mysql进行相关设置操作

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

mysql> SET GLOBAL group_replication_bootstrap_group=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (2.04 sec)

mysql> SET GLOBAL group_replication_bootstrap_group=OFF;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | ba65147e-5cb1-11ec-9e7e-005056864b71 | VZDTHMI023  |        3306 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
1 row in set (0.00 sec)
```

要保证上面的group_replication_applier的状态为"ONLINE"才对！

注意：标绿的两条命令，是为了标示以后加入集群的服务器以这台服务器为基准，以后加入的就不需要设置。用于第一次搭建MGR或重建MGR的时候使用，只需要在集群内的其中一台开启。

Node02服务器上操作

修改node2服务器的/etc/my.cnf配置文件

与node01相比，配置文件只有标亮的项目不同。

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
server_id = 2
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
loose-group_replication_local_address= "10.2.18.24:33061"
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

重启MySQL服务

```
[root@VZDTHBI024 ~]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

登录node02服务器进行操作

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

Node03服务器上操作

修改node3服务器的/etc/my.cnf配置文件

与node01相比，配置文件只有标亮的项目不同。

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
server_id = 3
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
loose-group_replication_local_address= "10.2.18.25:33061"
loose-group_replication_group_seeds= "10.2.18.23:33061,10.2.18.24:33061,10.2.18.25:33061"
loose-group_replication_bootstrap_group=off
```

重启MySQL服务

```
[root@VZDTHBI024 ~]# service mysql restart
Shutting down MySQL.. SUCCESS! 
Starting MySQL. SUCCESS!
```

登录node03服务器进行操作

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

至此，mgr集群已经配置完成。

### 配置ProxySQL服务

> 6032 是 ProxySQL 的管理端口号；
> 6033是对外服务的端口号。
> ProxySQL 的用户名和密码都是默认的 admin

修改MGR集群内所有服务器和proxysql服务器/etc/hosts文件

```
10.2.18.27  VZWTDZI027
10.2.18.24	VZDTHBI024
10.2.18.25	VZDTHMI025
10.2.18.22	VZDTHMI022
```

在proxy服务器上安装MySQL客户端

```
[root@VZDTHMI022 ~]# vim /etc/yum.repos.d/mariadb.repo
[mariadb]
name = MariaDB
baseurl = https://yum.mariadb.org/10.6.5/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

[root@VZDTHMI022 ~]# yum install -y MariaDB-client
```

安装proxysql

从GitHub下载最新的安装包https://github.com/sysown/proxysql/releases

https://github.com/sysown/proxysql/releases/download/v2.3.2/proxysql-2.3.2-1-centos7.x86_64.rpm

规划读写组

|默认的写组|	后备的写组|	读组|	离线组|
|-|-|-|-|
|10	|20	|30	|40|

首先安装一些依赖包

```
yum install -y perl-DBI perl-DBD-MySQL libgnutls* gnutls
```

安装proxysql的软件包

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

启动服务

tips

> 6032是ProxySQL的管理端口，所有对ProxySQL配置和管理工作都用这个端口登录；
> 6033是ProxySQL对外的服务端口，客户端通过这个端口访问后端的数据库。

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
proxysql数据库解释：

> • main库：是ProxySQL最主要的库，是需要修改配置时使用的库，它其实是一个内存数据库系统。所以，修改main库中的配置后，必须将其持久化到disk上才能永久保存。
> • disk库：是磁盘数据库，该数据库结构和内存数据库完全一致。当持久化内存数据库中的配置时，其实就是写入到disk库中。磁盘数据库的默认路径为$DATADIR/proxysql.db。
> • stats库：是统计信息库。这个库中的数据一般是在检索其内数据时临时填充的，它保存在内存中。因为没有相关的配置项，所以无需持久化。
> • monitor库：是监控后端MySQL节点相关的库，该库中只有几个log类的表，监控模块收集到的监控信息全都存放到对应的log表中。
> • stats_history库：是1.4.4版新增的库，用于存放历史统计数据。默认路径为$DATADIR/proxysql_stats.db。

初始化Proxysql，将之前的proxysql数据都删除(如果有)

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

在数据库端建立proxysql登入需要的帐号 （在三个MGR任意一个节点上操作，会自动同步到其他节点）(单主模式的话就在可写的MySQL节点执行)

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

创建检查MGR节点状态的函数和视图 
（在三个MGR任意一个节点上操作，会自动同步到其他节点）(单主模式的话就在可写的MySQL节点执行)
在MGR的主节点上，创建系统视图sys.gr_member_routing_candidate_status，该视图将为ProxySQL提供组复制相关的监控状态指标。

将addition_to_sys.sql脚本上传到MGR集群任意一个成员服务器的/opt目录下，执行如下语句导入MySQL即可 (在MGR的一个节点的mysql执行后，会同步到其他两个节点上)。

script for mysql 5.7

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

script for mysql 8.0

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

导入addition_to_sys.sql文件数据

```
[root@VZDTHMI023 opt]# mysql -uroot -p < /opt/addition_to_sys.sql
Enter password:
```

此时登录数据库就可以查看到这个视图了

```
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
```

将MGR节点写入proxysql的mysql_servers表中

这里设置hostgroup_id为10

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

表mysql_servers：ProxySQL连接到的后端服务器列表

配置mysql_group_replication_hostgroups表

```
insert into mysql_group_replication_hostgroups(writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup,active,max_writers,writer_is_also_reader,max_transactions_behind)  values(10,20,30,40,1,1,0,1000);

MySQL [(none)]> load mysql servers to runtime;
Query OK, 0 rows affected (0.004 sec)

MySQL [(none)]> save mysql servers to disk;
Query OK, 0 rows affected (0.020 sec)
```

然后，加入读写分离规则，让所有的SELECT操作路由到hostgroup_id为2的hostgroup:
设置读写分离（如果不配置，则所有读写操作均在主节点上。）
检查是否有配置过的读写分离规则（新配置的proxy不会有，如果之前有配置，则先删除掉。）

```
MySQL [(none)]> select * from mysql_query_rules;
Empty set (0.000 sec)
#如果有配置，则删除
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

```
为了便于学习和理解，查看以下mysql_query_rules表的结构。
MySQL [(none)]> show create table mysql_query_rules\G
*************************** 1. row ***************************
       table: mysql_query_rules
Create Table: CREATE TABLE mysql_query_rules (
    rule_id INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL,
    active INT CHECK (active IN (0,1)) NOT NULL DEFAULT 0,
    username VARCHAR,
    schemaname VARCHAR,
    flagIN INT CHECK (flagIN >= 0) NOT NULL DEFAULT 0,
    client_addr VARCHAR,
    proxy_addr VARCHAR,
    proxy_port INT CHECK (proxy_port >= 0 AND proxy_port <= 65535),
    digest VARCHAR,
    match_digest VARCHAR,
    match_pattern VARCHAR,
    negate_match_pattern INT CHECK (negate_match_pattern IN (0,1)) NOT NULL DEFAULT 0,
    re_modifiers VARCHAR DEFAULT 'CASELESS',
    flagOUT INT CHECK (flagOUT >= 0),
    replace_pattern VARCHAR CHECK(CASE WHEN replace_pattern IS NULL THEN 1 WHEN replace_pattern IS NOT NULL AND match_pattern IS NOT NULL THEN 1 ELSE 0 END),
    destination_hostgroup INT DEFAULT NULL,
    cache_ttl INT CHECK(cache_ttl > 0),
    cache_empty_result INT CHECK (cache_empty_result IN (0,1)) DEFAULT NULL,
    cache_timeout INT CHECK(cache_timeout >= 0),
    reconnect INT CHECK (reconnect IN (0,1)) DEFAULT NULL,
    timeout INT UNSIGNED CHECK (timeout >= 0),
    retries INT CHECK (retries>=0 AND retries <=1000),
    delay INT UNSIGNED CHECK (delay >=0),
    next_query_flagIN INT UNSIGNED,
    mirror_flagOUT INT UNSIGNED,
    mirror_hostgroup INT UNSIGNED,
    error_msg VARCHAR,
    OK_msg VARCHAR,
    sticky_conn INT CHECK (sticky_conn IN (0,1)),
    multiplex INT CHECK (multiplex IN (0,1,2)),
    gtid_from_hostgroup INT UNSIGNED,
    log INT CHECK (log IN (0,1)),
    apply INT CHECK(apply IN (0,1)) NOT NULL DEFAULT 0,
    attributes VARCHAR CHECK (JSON_VALID(attributes) OR attributes = '') NOT NULL DEFAULT '',
    comment VARCHAR)
1 row in set (0.000 sec)
```

```
mysql_query_rules字段属性解释
rule_id：规则的id。规则是按照rule_id的顺序进行处理的。
active：只有该字段值为1的规则才会加载到runtime数据结构，所以只有这些规则才会被查询处理模块处理。
username：用户名筛选，当设置为非NULL值时，只有匹配的用户建立的连接发出的查询才会被匹配。
schemaname：schema筛选，当设置为非NULL值时，只有当连接使用schemaname作为默认schema时，该连接发出的查询才会被匹配。(在MariaDB/MySQL中，schemaname等价于databasename)。
flagIN,flagOUT：这些字段允许我们创建"链式规则"(chains of rules)，一个规则接一个规则。
apply：当匹配到该规则时，立即应用该规则。
client_addr：通过源地址进行匹配。
proxy_addr：当流入的查询是在本地某地址上时，将匹配。
proxy_port：当流入的查询是在本地某端口上时，将匹配。
digest：通过digest进行匹配，digest的值在stats_mysql_query_digest.digest中。
match_digest：通过正则表达式匹配digest。
match_pattern：通过正则表达式匹配查询语句的文本内容。
negate_match_pattern：设置为1时，表示未被match_digest或match_pattern匹配的才算被成功匹配。也就是说，相当于在这两个匹配动作前加了NOT操作符进行取反。
re_modifiers：RE正则引擎的修饰符列表，多个修饰符使用逗号分隔。指定了CASELESS后，将忽略大小写。指定了GLOBAL后，将替换全局(而不是第一个被匹配到的内容)。为了向后兼容，默认只启用了CASELESS修饰符。
replace_pattern：将匹配到的内容替换为此字段值。它使用的是RE2正则引擎的Replace。注意，这是可选的，当未设置该字段，查询处理器将不会重写语句，只会缓存、路由以及设置其它参数。
destination_hostgroup：将匹配到的查询路由到该主机组。但注意，如果用户的transaction_persistent=1(见mysql_users表)，且该用户建立的连接开启了一个事务，则这个事务内的所有语句都将路由到同一主机组，无视匹配规则。
cache_ttl：查询结果缓存的时间长度(单位毫秒)。注意，在ProxySQL 1.1中，cache_ttl的单位是秒。
reconnect：目前不使用该功能。
timeout：被匹配或被重写的查询执行的最大超时时长(单位毫秒)。如果一个查询执行的时间太久(超过了这个值)，该查询将自动被杀掉。如果未设置该值，将使用全局变量mysql-default_query_timeout的值。
retries：当在执行查询时探测到故障后，重新执行查询的最大次数。如果未指定，则使用全局变量mysql-query_retries_on_failure的值。
delay：延迟执行该查询的毫秒数。本质上是一个限流机制和QoS，使得可以将优先级让位于其它查询。这个值会写入到mysql-default_query_delay全局变量中，所以它会应用于所有的查询。将来的版本中将会提供一个更高级的限流机制。
mirror_flagOUT和mirror_hostgroup：mirroring相关的设置，目前mirroring正处于实验阶段，所以不解释。
error_msg：查询将被阻塞，然后向客户端返回error_msg指定的信息。
sticky_conn：当前还未实现该功能。
multiplex：如果设置为0，将禁用multiplexing。如果设置为1，则启用或重新启用multiplexing，除非有其它条件(如用户变量或事务)阻止启用。如果设置为2，则只对当前查询不禁用multiplexing。默认值为NULL，表示不会修改multiplexing的策略。
log：查询将记录日志。
apply：当设置为1后，当匹配到该规则后，将立即应用该规则，不会再评估其它的规则(注意：应用之后，将不会评估mysql_query_rules_fast_routing中的规则)。
comment：注释说明字段，例如描述规则的意义。

解释说明:
match_pattern的规则是基于正则表达式的，
active表示是否启用这个sql路由项，
match_pattern就是我们正则匹配项，
destination_hostgroup表示我们要将该类sql转发到哪些mysql上面去，这里我们将select转发到group 2，。
apply为1表示该正则匹配后，将不再接受其他匹配，直接转发。

```

查看MGR配置

```
MySQL [(none)]> select * from mysql_group_replication_hostgroups\G
*************************** 1. row ***************************
       writer_hostgroup: 10  # 写组
backup_writer_hostgroup: 20  # 后备写组
       reader_hostgroup: 30  # 读组
      offline_hostgroup: 40  # 下线组
                 active: 1   # 是否启用
            max_writers: 1   # 最多的写节点个数
  writer_is_also_reader: 0   # 决定一个节点升级为写节点(放进writer_hostgroup)后是否仍然保留在reader_hostgroup组中提供读服务。如果MGR多主模式需要设置为1
max_transactions_behind: 1000   #  该字段决定最多延后写节点多少个事务
                comment: NULL # 注释
1 row in set (0.000 sec)
```

在proxysql中增加帐号

表MySQL_users：连接到ProxySQL的用户及其凭据列表。 请注意，ProxySQL也将使用相同的凭据连接到后端服务器！

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

特别注意，不能使用mysql_users里面的用户来做监控用户。

为了避免出现“9006 - ProxySQL Error: connection is locked to hostgroup 1 but trying to reach hostgroup 2”的错误，设置proxysql如下变量：

```
MySQL [(none)]> set mysql-set_query_lock_on_hostgroup=0; #set语法兼容
Query OK, 1 row affected (0.000 sec)

MySQL [(none)]> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.002 sec)

MySQL [(none)]> SAVE MYSQL VARIABLES TO DISK;
Query OK, 147 rows affected (0.004 sec)
```

> mysql-set_query_lock_on_hostgroup
> 激活时（从 2.0.6 开始默认），如果在多语句命令中使用 SET 语句或如果 SET 语句解析不成功，则禁用多路复用和查询路由。客户端将保持绑定到单个后端连接。任何 ProxySQL 不理解的 SET 语句都将禁用多路复用和路由。

最后需要将global_variables，mysql_servers、mysql_users表的信息加载到RUNTIME，更进一步加载到DISK:

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

在MGR集群中增加监控用户（monitor）

```
create user monitor@'%' identified by 'monitor';
grant usage,replication client on *.* to monitor@'%';
grant select on sys.* to monitor;
flush privileges;
```

验证proxysql登录

```
[root@VZDTHMI022 etc]# mysql -uproxysql -pproxysql -h 127.0.0.1 -P6033 -e"select @@hostname"
+------------+
| @@hostname |
+------------+
| VZDTHMI025 |
+------------+
```

配置proxysql web界面监控

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

https://x.x.x.x:6080/ 

账号 密码都是 stats

这个页面只能供查看，不能修改proxysql的运行配置。

配置proxysql集群

在单proxysql节点的基础上再加一个proxysql节点

设置集群管理账号（在所有proxysql节点上执行）

```
update global_variables set variable_value='admin:admin;cluster:cluster' where variable_name='admin-admin_credentials';
update global_variables set variable_value='cluster' where variable_name='admin-cluster_username';
update global_variables set variable_value='cluster' where variable_name='admin-cluster_password';

```

设置集群成员（在所有proxysql节点上执行）

```
insert into proxysql_servers(hostname,port,weight,comment) values('10.2.18.22',6032,1,'primary'),('10.2.18.30',6032,1,'secondary');
```

加载配置
先加载已有后端服务器配置的第一个ProxySQL节点（原proxysql节点），再加载其他新节点。

```
load admin variables to runtime;
load proxysql servers to runtime;
save admin variables to disk;
save proxysql servers to disk;
```

查看群集状态

```
select hostname,port,comment,Uptime_s,last_check_ms from stats_proxysql_servers_metrics;
select hostname,name,checksum,updated_at from stats_proxysql_servers_checksums;
```

此时新加入的proxysql节点已经从原来的proxysql同步到了配置数据，比如后端服务器（mysql_servers），用户（mysql_users）等。

配置proxysql集群的自动故障转移（keepalived）

在两个proxysql节点上分别安装keepalived软件

```
yum install -y keepalived
```

安装完后备份一次原始的配置文件

```
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
```

创建一个检测proxysql的检测脚本

```
touch /etc/keepalived/check_proxysql.sh

文件内容：
#!/bin/bash
/usr/bin/netstat -na | grep -e '0.0.0.0:6033' -e '0.0.0.0:6032' &>/dev/null

```

安装net-tools工具包

```
yum install net-tools
```

然后开始修改配置文件
vi /etc/keepalived/keepalived.conf
由于配置的proxysql1的keepalived中的priority （优先级）高于proxysql2的优先级，所以，在proxysql1故障并恢复后，会强制重新接管服务。

**配置文件内容（proxysql1）**

```
! Configuration File for keepalived

global_defs {
   router_id proxysql_ha
}

#定义一个检测proxysql是否存活的脚本
vrrp_script chk_proxysql_port {
    # 调用的检测proxysql脚本文件
    script "/etc/keepalived/check_proxysql.sh"

    # 脚本执行间隔，每1s检测一次
    interval 1

    # 脚本结果导致的优先级变更，检测失败（脚本返回非0）则优先级减50
    weight -50

    #检测连续2次失败才算确定是真失败。会用weight减少优先级（1-255之间）
    fall 2

    #检测2次成功就算成功。但不修改优先级
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

#定义一个检测proxysql是否存活的脚本
vrrp_script chk_proxysql_port {
    # 调用的检测proxysql脚本文件
    script "/etc/keepalived/check_proxysql.sh"

    # 脚本执行间隔，每1s检测一次
    interval 1

    # 脚本结果导致的优先级变更，检测失败（脚本返回非0）则优先级减50
    weight -50

    #检测连续2次失败才算确定是真失败。会用weight减少优先级（1-255之间）
    fall 2

    #检测2次成功就算成功。但不修改优先级
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

如果出现两个节点同时挂载到了虚拟IP的情况，添加下面的防火墙规则：

```
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface ens192 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --reload
```
