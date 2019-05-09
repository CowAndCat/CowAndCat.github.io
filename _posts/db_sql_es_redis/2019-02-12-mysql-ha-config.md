---
layout: post
title: 高可用MySQL配置方法（主从）
category: mysql
comments: false
---

## 零、写在前面

本文将介绍三种配置高可用MySQL的方法，分别应对单机、多机器、Docker的部署环境。本文采用的方案均为一主二从架构。

## 一、单机器

### 1.1 Mac下安装多个MySQL实例

首先创建三个文件夹用来存储主备数据；之后执行以下初始化命令，加上--initialize-insecure参数则生成的root用户没有密码，否则mysql初始化时随机生成一个密码并输入到日志文件中

    mysqld --datadir=/Users/maofagui/programF/mysql/master/data  --initialize --initialize-insecure
    mysqld --datadir=/Users/maofagui/programF/mysql/slave/data  --initialize --initialize-insecure
    mysqld --datadir=/Users/maofagui/programF/mysql/slave1/data  --initialize --initialize-insecure

由于要启动多个实例，需要用到mysqld_multi命令，多个数据库实例公用一个配置文件，保存为cluster.conf，如下所示：

    [mysqld_multi]
    mysqld     = /usr/local/opt/mysql@5.7/bin/mysqld
    mysqladmin = /usr/local/opt/mysql@5.7/bin/mysqladmin
    user       = root
    password   = root


    [mysqld3307]
    server-id=3307
    port=3307
    log-bin=mysql-bin

    log-error=/Users/maofagui/programF/mysql/master/mysqld.log
    tmpdir=/Users/maofagui/programF/mysql/master
    slow_query_log=on
    slow_query_log_file =/Users/maofagui/programF/mysql/master/mysql-slow.log
    long_query_time=1

    socket=/Users/maofagui/programF/mysql/master/mysql_3307.sock
    pid-file=/Users/maofagui/programF/mysql/master/mysql.pid

    basedir=/Users/maofagui/programF/mysql/master
    datadir=/Users/maofagui/programF/mysql/master/data

    [mysqld3308]
    server-id=3308
    port=3308
    log-bin=mysql-bin

    log-error=/Users/maofagui/programF/mysql/slave/mysqld.log
    tmpdir=/Users/maofagui/programF/mysql/slave

    slow_query_log=on
    slow_query_log_file =/Users/maofagui/programF/mysql/slave/mysql-slow.log
    long_query_time=1

    socket=/Users/maofagui/programF/mysql/slave/mysql_3308.sock
    pid-file=/Users/maofagui/programF/mysql/slave/mysql.pid


    basedir=/Users/maofagui/programF/mysql/slave
    datadir=/Users/maofagui/programF/mysql/slave/data

    read_only=1

    [mysqld3309]
    server-id=3309
    port=3309
    log-bin=mysql-bin

    log-error=/Users/maofagui/programF/mysql/slave1/mysqld.log
    tmpdir=/Users/maofagui/programF/mysql/slave1

    slow_query_log=on
    slow_query_log_file =/Users/maofagui/programF/mysql/slave1/mysql-slow.log
    long_query_time=1

    socket=/Users/maofagui/programF/mysql/slave1/mysql_3308.sock
    pid-file=/Users/maofagui/programF/mysql/slave1/mysql.pid


    basedir=/Users/maofagui/programF/mysql/slave1
    datadir=/Users/maofagui/programF/mysql/slave1/data

    read_only=1

    [mysqld]
    character_set_server=utf8

接下来，运行`mysqld_multi --defaults-file=cluster.conf start`即可启动所有mysql实例:
    
    $ mysqld_multi --defaults-file=cluster.conf start
    WARNING: Log file disabled. Maybe directory or file isn't writable?
    mysqld_multi log file version 2.16; run: 二  2 12 16:50:27 2019

    Starting MySQL servers

### 1.2 配置主从复制

连接上主mysql，创建用于复制的用户并赋予复制权限：

    $mysql -hlocalhost -uroot -P3307
    mysql> create user 'copy'@'%' identified by 'copy';
    Query OK, 0 rows affected (0.07 sec)

    mysql> grant replication slave on *.* to 'copy'@'%';
    Query OK, 0 rows affected (0.02 sec)

执行show master status命令

    mysql> show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000002 |     1166 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+


连接从mysql，设置master信息并启动slave：

    $mysql -hlocalhost -uroot -P3308
    mysql> change master to master_host='127.0.0.1', master_port=3307,master_user='copy',master_password='copy',master_log_file='mysql-bin.000002',master_log_pos=1166;
    Query OK, 0 rows affected, 1 warning (0.10 sec)

    mysql> start slave;
    Query OK, 0 rows affected (0.01 sec)

执行show slave status命令，查看slave状态，发现Slave_IO_State是Waiting for master to send event，Slave_IO_Running和Slave_SQL_Running状态是YES就说明设置成功。

    mysql> show slave status\G;
    *************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 127.0.0.1
                  Master_User: copy
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1166
               Relay_Log_File: 192-relay-bin.000002
                Relay_Log_Pos: 322
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes

### 1.3 mysqld开闭命令

    开启  sudo /usr/local/mysql/support-files/mysql.server start
    关闭  sudo /usr/local/mysql/support-files/mysql.server stop
    或   kill -9 PID 
    这里的PID是mysqld_safe进程的ID。


## 二、多机器

这里基于百度BCC来进行部署。

### 2.1 主从配置
先安装好MySQL server和client（见[mysql 安装](/mysql/2017/04/14/mysql-install.html)）。然后开始配置。

主库(192.168.16.103)上的my.cnf配置:
    
    # For advice on how to change settings please see
    # http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html

    [mysqld]

    # Remove leading # and set to the amount of RAM for the most important data
    # cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
    # innodb_buffer_pool_size = 128M

    # Remove leading # to turn on a very important data integrity option: logging
    # changes to the binary log between backups.
    log_bin = mysql-bin

    # These are commonly set, remove the # and set as required.
    #basedir=/usr/local/mysql/master
    #datadir=/usr/local/mysql/master/data
    #socket=/usr/local/mysql/master/mysql.sock

    port = 3307
    server_id = 103

    # Remove leading # to set options mainly useful for reporting servers.
    # The server defaults are faster for transactions and fast SELECTs.
    # Adjust sizes as needed, experiment to find the optimal values.
    # join_buffer_size = 128M
    # sort_buffer_size = 2M
    # read_rnd_buffer_size = 2M

    #sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

重启mysql：`/etc/init.d/mysql restart`

连接mysql：`mysql -hlocalhost -uroot -p`

    mysql> show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000003 |      262 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    mysql>create user repl; 
    //创建新用户
    //repl用户必须具有REPLICATION SLAVE权限，除此之外没有必要添加其他权限，密码为123456。
    //说明一下192.168.0.%，这个配置是指明repl用户所在服务器，这里%是通配符，表示192.168.0.0-192.168.0.255的Server都可以以repl用户登陆主服务器。当然你也可以指定固定Ip。
    mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.%.%' IDENTIFIED BY '123456';

配置从库(192.168.16.102/192.168.16.104),my.cnf：

    # For advice on how to change settings please see
    # http://dev.mysql.com/doc/refman/5.6/en/server-configuration-defaults.html

    [mysqld]

    # Remove leading # and set to the amount of RAM for the most important data
    # cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
    # innodb_buffer_pool_size = 128M

    # Remove leading # to turn on a very important data integrity option: logging
    # changes to the binary log between backups.
    log_bin = mysql-bin

    # These are commonly set, remove the # and set as required.
    #basedir=/usr/local/mysql/master
    #datadir=/usr/local/mysql/master/data
    #socket=/usr/local/mysql/master/mysql.sock

    port = 3307
    server_id = 102 #104
    read_only = 1
    log_slave_updates=1

    # Remove leading # to set options mainly useful for reporting servers.
    # The server defaults are faster for transactions and fast SELECTs.
    # Adjust sizes as needed, experiment to find the optimal values.
    # join_buffer_size = 128M
    # sort_buffer_size = 2M
    # read_rnd_buffer_size = 2M

    #sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES

配置relay_log（指定中继日志的位置和命名，可以用系统默认值）和log_slave_updates（允许备库将其重放的事件也记录到自身的二进制日志中，默认为关闭）

read_only配置选项会阻止任何没有特权权限的线程修改数据（所以最好不要给予用户超出需要的权限）。但read_only选项常常不是很实用，特别是对于那些需要在备库建表的应用。

重启,连接mysql：`mysql -hlocalhost -uroot -p`

    mysql> change master to 
    master_host='192.168.16.103',
    master_port=3307,
    master_user='repl',
    master_password='123456',
    master_log_file='mysql-bin.000004',
    master_log_pos=120;

    mysql> set global read_only=1; // 注意切换到root用户后还是可以进行增删改查的。

    mysql> start slave;
    Query OK, 0 rows affected (0.00 sec)

    mysql> show slave status \G;
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 192.168.16.103
                      Master_User: repl
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000004
              Read_Master_Log_Pos: 120
                   Relay_Log_File: instance-3d7aof0v-2-relay-bin.000002
                    Relay_Log_Pos: 283
            Relay_Master_Log_File: mysql-bin.000004
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB:
              Replicate_Ignore_DB:
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                       Last_Errno: 0
                       Last_Error:
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 120
                  Relay_Log_Space: 470
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 0
                   Last_SQL_Error:
      Replicate_Ignore_Server_Ids:
                 Master_Server_Id: 103
                      Master_UUID: d5c01981-2f35-11e9-96b5-fa163efe1252
                 Master_Info_File: /var/lib/mysql/master.info
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
               Master_Retry_Count: 86400
                      Master_Bind:
          Last_IO_Error_Timestamp:
         Last_SQL_Error_Timestamp:
                   Master_SSL_Crl:
               Master_SSL_Crlpath:
               Retrieved_Gtid_Set:
                Executed_Gtid_Set:
                    Auto_Position: 0
    1 row in set (0.00 sec)


下面进行测试：

在主库上创建表：

    mysql> create table tt(`id` int(11) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(10) NOT NULL,
    PRIMARY KEY(`id`));

    mysql> show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000004 |      303 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

到从库上能看到：

    mysql> show tables;
    +----------------+
    | Tables_in_test |
    +----------------+
    | tt             |
    +----------------+

    mysql> show slave status \G;
    // 能看到 Exec_Master_Log_Pos: 303

### 2.2 主备不同步的解决

直接用root账户登录102或104机器，插入一条数据，然后在登录103主库，插入一条数据，这时会导致主备数据冲突。

    mysql> show slave status \G;
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 192.168.16.103
                      Master_User: repl
                      Master_Port: 3307
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.000004
              Read_Master_Log_Pos: 557
                   Relay_Log_File: instance-3d7aof0v-1-relay-bin.000004
                    Relay_Log_Pos: 283
            Relay_Master_Log_File: mysql-bin.000004
                 Slave_IO_Running: Yes
                Slave_SQL_Running: No
                  Replicate_Do_DB:
              Replicate_Ignore_DB:
               Replicate_Do_Table:
           Replicate_Ignore_Table:
          Replicate_Wild_Do_Table:
      Replicate_Wild_Ignore_Table:
                       Last_Errno: 1062
                       Last_Error: Error 'Duplicate entry '1' for key 'PRIMARY'' on query. Default database: 'test'. Query: 'insert into tt(`name`) values ('abcd')'
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 303
                  Relay_Log_Space: 724
                  Until_Condition: None
                   Until_Log_File:
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File:
               Master_SSL_CA_Path:
                  Master_SSL_Cert:
                Master_SSL_Cipher:
                   Master_SSL_Key:
            Seconds_Behind_Master: NULL
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error:
                   Last_SQL_Errno: 1062
                   Last_SQL_Error: Error 'Duplicate entry '1' for key 'PRIMARY'' on query. Default database: 'test'. Query: 'insert into tt(`name`) values ('abcd')'
      Replicate_Ignore_Server_Ids:
                 Master_Server_Id: 103
                      Master_UUID: d5c01981-2f35-11e9-96b5-fa163efe1252
                 Master_Info_File: /var/lib/mysql/master.info
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State:
               Master_Retry_Count: 86400
                      Master_Bind:
          Last_IO_Error_Timestamp:
         Last_SQL_Error_Timestamp: 190213 12:10:08

Slave_SQL_Running: No 说明slave已经拒绝同步了。

解决办法一：忽略错误后，继续同步。

该方法适用于主从库数据相差不大，或者要求数据可以不完全统一的情况，数据要求不严格的情况。

    mysql> stop slave; 
    #表示跳过一步错误，后面的数字可变 
    mysql> set global sql_slave_skip_counter =1; 
    mysql> start slave; 
    mysql> show slave status\G;

这个模式，如果遇到错误，会忽略然后继续进行同步，对于错误（比如数据冲突）会不进行处理。

解决办法二：重新做主从，完全同步。

该方法适用于主从库数据相差较大，或者要求数据完全统一的情况 。

解决步骤如下： 

1.先进入主库，进行锁表，防止数据写入 
    
    mysql> show processlist;
    mysql> flush tables with read lock; 
注意：该处是锁定为只读状态，语句不区分大小写.

该命令关闭所有打开的表，同时对于所有数据库中的表都加一个读锁，直到显示地执行unlock tables，该操作常常用于数据备份的时候。也就是将所有的脏页都要刷新到磁盘，然后对所有的表加上了读锁，于是这时候直接拷贝数据文件也就是安全的。

2.进行数据备份 

    [root@server01 mysql]# mysqldump -uroot -p -hlocalhost test > mysql.bak.sql 
    #把数据备份到mysql.bak.sql文件 

这里注意一点：数据库备份一定要定期进行，可以用shell脚本或者python脚本，都比较方便，确保数据万无一失。

3.查看master 状态 

    mysql> show master status; 
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000004 |     2050 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec) 

4.把mysql备份文件传到从库机器，进行数据恢复 

    #使用scp命令 
    [root@server01 mysql]# scp mysql.bak.sql root@192.168.16.103:/root

5.停止从库的状态 

    mysql> stop slave; 

6.然后到从库执行mysql命令，导入数据备份 
    
    mysql> source /root/mysql.bak.sql 

7.设置从库同步，注意该处的同步点，就是主库show master status信息里的`File`、`Position`两项 

    change master to 
    master_host='192.168.16.103',
    master_port=3307,
    master_user='repl',
    master_password='123456',
    master_log_file='mysql-bin.000004',
    master_log_pos=2050;

8.重新开启从同步 
    
    mysql> start slave; 

9.查看同步状态 

    mysql> show slave status\G 
    查看： 
    Slave_IO_Running: Yes 
    Slave_SQL_Running: Yes 

10.解除主库的锁定 

    mysql> unlock tables;

### 2.3 主从切换

在开始切换之前先对主库进行锁表：

    mysql> flush tables with read lock;

1）在master执行：show processlist;  
显示 Master has sent all binlog to slave; waiting for binlog to be updated

2）在slave执行：

    mysql> show processlist；  
    #显示 Slave has read all relay log; waiting for the slave I/O thread to update it
    mysql> show slave status \G;  

检查IO及SQL线程是否正常，如果为NO表明同步不一致，需要重新将slave同步保持主从数据一致。

3）停止slave io线程  
在slave执行：

    mysql> STOP SLAVE IO_THREAD;
    mysql> SHOW PROCESSLIST;

确保状态为：Slave has read all relay log

以上都执行完成后可以把slave提升为master：

4）提升slave为master

    mysql> stop slave;
    mysql> Reset master;
    mysql> Reset slave all; #在5.6.3版本之后, 在5.6.3版本之前是Reset slave; 

查看slave是否只读模式：`show variables like 'read_only';`

只读模式需要修改my.cnf文件，注释read-only=1并重启mysql服务。

或者不重启使用命令关闭只读，但下次重启后失效：`set global read_only=off;`

    mysql> show master status \G;
    mysql> show slave status \G;

备注：reset slave all 命令会删除从库的 replication 参数，之后 show slave status\G 的信息返回为空。

5）将原来master变为slave

在新的master上创建同步用户：

    create user repl;
    GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.%.%' IDENTIFIED BY '123456';

在新的slave上重置binlog：

    mysql> Reset master;

    mysql> change master to master_host='192.168.16.102', 
    master_port=3307,
    master_user='repl',
    master_password='123456',
    master_log_file='master-bin.000001',
    master_log_pos=415;

以上最后两步可以在master执行：show master status, 启动slave：start slave; 并查看slave状态：show slave status\G;
将锁释放： unlock tables;

#### 2.3.1 slave 104遇到错误
Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'

【原因】 

该错误发生在从库的io进程从主库拉取日志时，发现主库的mysql_bin.index文件中第一个文件不存在。出现此类报错可能是由于你的slave 由于某种原因停止了好长一段是时间，当你重启slave 复制的时候，在主库上找不到相应的binlog ,会报此类错误。或者是由于某些设置主库上的binlog被删除了，导致从库获取不到对应的binglog file。

【如何解决】  

1.`stop slave; change master to master_log_file='mysql-bin.000001',master_log_pos=415; start slave;`

如果不行，再重新搭建slave。  
2 注意主库binlog的清理策略，选择基于时间过期的删除方式还是基于空间利用率的删除方式。  
  不要使用rm -fr 命令删除binlog file，这样不会同步修改mysql_bin.index 记录的binlog 条目。在删除binlog的时候确保主库保留了从库 show slave status 的Relay_Master_Log_File对应的binlog file。

### 2.4 增加备库

方法1. 使用冷备份：最基本的方法是关闭主库，把数据复制到备库（高效复制文件的方法参考附录C）。重启主库后，会使用一个新的二进制日志文件，我们在备库通过执行CHANGE MASTER TO指向这个文件的起始处。这个方法的缺点很明显：在复制数据时需要关闭主库。

方法2. 使用mysqldump：可以使用以下命令来转储主库数据并将其加载到备库，然后设置相应的二进制日志坐标：

    $ mysqldump --single-transaction --all-databases --master-data=1 --host=server1 | mysql --host=server2

选项--single-transaction使得转储的数据为事务开始前的数据。如果使用的是非事务型表，可以使用--lock-all-tables选项来获得所有表的一致性转储。

方法3. 使用快照或备份  
只要知道对应的二进制日志坐标，就可以使用主库的快照或者备份来初始化备库（如果使用备份，需要确保从备份的时间点开始的主库二进制日志都要存在）。只需要把备份或快照恢复到备库，然后使用CHANGE MASTER TO指定二进制日志的坐标。也可以使用LVM快照、SAN快照、EBS快照——任何快照都可以。

方法4. 使用Percona Toolkit里的pt-table-sync。

### 2.5 复制模式的选择

复制的实现有两种方式：基于语句的和基于行的。

如果有多层的复制服务器，并且所有的都被配置成基于行的复制模式，当会话级别的变量@@binlog_format被设置成STATEMENT时，所执行的语句在源服务器上被记录为基于语句的模式，但第一层的备库可能将其记录成行模式，并传递给其他层的备库。也就是说你期望的基于语句的日志在复制拓扑中将会被切换到基于行的模式。基于行的日志无法处理诸如在备库修改表的schema这样的情况，而基于语句的日志可以。

    mysql> show variables like '%binlog_format%';
    +---------------+-----------+
    | Variable_name | Value     |
    +---------------+-----------+
    | binlog_format | STATEMENT |
    +---------------+-----------+
    1 row in set (0.00 sec)

其他情况用默认的就行。

### 2.6 一些命令

    SHOW master logs; // 查看二进制文件和大小
    SHOW BINLOG EVENTS; // 查看复制事件
    SHOW BINLOG EVENTS IN 'mysql-bin.000223' FROM 13634\G 

## 3. 基于Docker的主从复制

### 3.1 搭建Docker服务器
首先拉取docker镜像：

    docker pull mysql:5.7

然后使用此镜像启动容器，这里需要分别启动主从两个容器

Master(主)：

    docker run -p 3339:3306 --name mysql_master -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

Slave(从)：

    docker run -p 3340:3306 --name mysql_slave -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

使用`docker ps`命令查看正在运行的容器

    shell> docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
    be5c00493437        mysql:5.7           "docker-entrypoint.s…"   6 seconds ago       Up 5 seconds        33060/tcp, 0.0.0.0:3340->3306/tcp   mysql_slave
    62ffedc13b4b        mysql:5.7           "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        33060/tcp, 0.0.0.0:3339->3306/tcp   mysql_master

连接测试：`mysql -h0.0.0.0 -P3340 -uroot -p`

### 3.2 配置Master(主)
通过`docker exec -it be5c00493437 /bin/bash`命令进入到Master容器内部，也可以通过`docker exec -it mysql_master /bin/bash`命令进入。be5c00493437是容器的id,而mymysql是容器的名称。

切换到/etc/mysql目录下，在docker容器内部自行安装vim,然后vi my.cnf对my.cnf进行编辑。

    hostname -i #记住ip，后面change master to 时要用到。
    apt-get update
    apt-get install vim    
    cd /etc/mysql
    vim my.cnf

在my.cnf中添加如下配置：

    [mysqld]
    ## 同一局域网内注意要唯一
    server-id=100  
    ## 开启二进制日志功能，可以随便取（关键）
    log-bin=mysql-bin

配置完成之后，需要重启mysql服务使配置生效。使用`service mysql restart`完成重启。重启mysql服务时会使得docker容器停止，我们还需要`docker start mysql_master`启动容器。

下一步在Master数据库创建数据同步用户，授予用户 slave REPLICATION SLAVE权限和REPLICATION CLIENT权限，用于在主从库之间同步数据。

    CREATE USER 'repl'@'%' IDENTIFIED BY '123456';
    GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';

### 3.3 配置Slave(从)
和配置Master(主)一样，在Slave配置文件my.cnf中添加如下配置：

    [mysqld]
    ## 设置server_id,注意要唯一
    server-id=101  
    ## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
    log-bin=mysql-slave-bin   
    ## relay_log配置中继日志
    relay_log=vio-mysql-relay-bin 
    readonly = 1

配置完成后也需要重启mysql服务和docker容器，操作和配置Master(主)一致。

### 3.4 链接Master(主)和Slave(从)
在Master进入mysql，执行show master status;

在Slave 中进入 mysql，执行

    mysql> change master to master_host='172.17.0.2',
    master_user='repl',
    master_password='123456',
    master_port=3306,
    master_log_file='mysql-bin.000001',
    master_log_pos=615, 
    master_connect_retry=30;

    mysql> start slave;
    mysql> show slave status \G;

注意，因为mysql master是运行在docker里面，所以不能这样设置：master_host='0.0.0.0'

## REF
> [MySQL的主从复制](https://www.linuxidc.com/Linux/2013-10/91682.htm)  
> [mac下安装多个mysql实例](https://www.cnblogs.com/wkzhao/p/10293127.html)  
> [Mysql主从（主从不同步解决办法，常见问题及解决办法，在线对mysql做主从复制）](http://blog.51cto.com/13407306/2067333)  
> [基于Docker的Mysql主从复制搭建](https://www.cnblogs.com/songwenjie/p/9371422.html)  
> [从零开始，通过docker实现mysql 主从复制，主主复制](https://blog.csdn.net/fyihdg/article/details/78951357)