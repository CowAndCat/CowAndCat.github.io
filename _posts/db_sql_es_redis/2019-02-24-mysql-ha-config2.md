---
layout: post
title: 高可用MySQL方案：MHA
category: mysql
comments: false
---
之前整理里两篇关于高可用MySQL的文章，其中的架构都是手动搭建，对于复制延迟和主从切换都需要额外的开发工作。本文将用一个新的MHA方案来简化手动操作的工作。

# 一、MHA

## 1.1 关于MHA
MHA（Master HA）是一款开源的MySQL的高可用程序，它为MySQL主从复制架构提供了automating master failover 功能。MHA在监控到master节点故障时，会提升其中拥有最新数据的slave节点成为新的master节点，在此期间，MHA会通过与其它从节点获取额外信息来避免一致性方面的问题。MHA还提供了master节点的在线切换功能，即按需切换master/slave节点。

相较于其它HA软件，MHA的目的在于维持MySQL Replication中Master库的高可用性，其最大特点是可以修复多个Slave之间的差异日志，最终使所有Slave保持数据一致，然后从中选择一个充当新的Master，并将其它Slave指向它。

## 1.2 MHA角色部署
MHA 服务有两种角色，MHA Manager（管理节点）和MHA Node（数据节点）：

- MHA Manager：通常单独部署在一台独立的机器上或者直接部署在其中一台slave上（不建议后者），管理多个master/slave集群，每个master/slave集群称作一个application；其作用有二：

    - （1）master自动切换及故障转移命令运行
    - （2）其他的帮助脚本运行：手动切换master；master/slave状态检测

- MHA node：运行在每台MySQL服务器上（master/slave/manager），它通过监控具备解析和清理logs功能的脚本来加快故障转移。其作用有：

    - （1）复制主节点的binlog数据
    - （2）对比从节点的中继日志文件
    - （3）无需停止从节点的SQL线程，定时删除中继日志
 
目前MHA主要支持一主多从的架构，要搭建MHA,要求一个复制集群中必须最少有三台数据库服务器，一主二从，即一台充当master，一台充当备用master，另外一台充当从库。

## 1.3 工作原理

- （1）从宕机崩溃的master保存二进制日志事件（binlog events）;
- （2）识别含有最新更新的slave；
- （3）应用差异的中继日志（relay log）到其他的slave；
- （4）应用从master保存的二进制日志事件（binlog events）；
- （5）提升一个slave为新的master；
- （6）使其他的slave连接新的master进行复制；

# 二、基于Docker的部署

## 2.1 Docker镜像制作

预置操作:

(opt)去官网下载manager和node安装包：[https://code.google.com/archive/p/mysql-master-ha/downloads](https://code.google.com/archive/p/mysql-master-ha/downloads)

- [mha4mysql-manager-0.55.tar.gz](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-manager-0.55.tar.gz)
- [mha4mysql-node-0.54.tar.gz](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-node-0.54.tar.gz)
- [mha4mysql-node-0.54-1.el5.noarch.rpm](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-node-0.54-1.el5.noarch.rpm)
- [mha4mysql-manager-0.55-1.el5.noarch.rpm](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-manager-0.55-1.el5.noarch.rpm)
- [mha4mysql-manager_0.55-0_all.deb](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-manager_0.55-0_all.deb)
- [mha4mysql-node_0.54-0_all.deb](https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/mysql-master-ha/mha4mysql-node_0.54-0_all.deb)

purge_relay_log.sh脚本内容：

    #!/bin/bash
    user=root
    passwd=123456
    port=3306
    log_dir='/data/masterha/log'
    work_dir='/data'
    purge='/usr/local/bin/purge_relay_logs'

    if [ ! -d $log_dir ]
    then
       mkdir $log_dir -p
    fi

    $purge --user=$user --password=$passwd --disable_relay_log_purge --port=$port --workdir=$work_dir >> $log_dir/purge_relay_logs.log 2>&1

(opt) 直接用现成的docker镜像：https://hub.docker.com/r/breeze2/mha4mysql-manager、https://hub.docker.com/r/breeze2/mha4mysql-node

node的Dockerfile内容如下：

    FROM mysql:5.6

    COPY ./mha4mysql-node.tar.gz /tmp/

    RUN build_deps='ssh sshpass perl libdbi-perl libmodule-install-perl libdbd-mysql-perl make' \
        && apt-get update \
        && apt-get -y --force-yes install $build_deps \
        && tar -zxf /tmp/mha4mysql-node.tar.gz -C /opt \
        && cd /opt/mha4mysql-node \
        && perl Makefile.PL \
        && make \
        && make install \
        && cd /opt \
        && rm -rf /opt/mha4mysql-* \
        && apt-get clean

> sshpass 是一个简单、轻量级的命令行工具，通过它我们能够向命令提示符本身提供密码（非交互式密码验证）

manager的Dockerfile内容如下：

    FROM debian:jessie

    COPY ./mha4mysql-manager.tar.gz /tmp/
    COPY ./mha4mysql-node.tar.gz /tmp/

    RUN build_deps='ssh sshpass perl libdbi-perl libmodule-install-perl libdbd-mysql-perl libconfig-tiny-perl liblog-dispatch-perl libparallel-forkmanager-perl make' \
        && apt-get update \
        && apt-get -y --force-yes install $build_deps \
        && tar -zxf /tmp/mha4mysql-node.tar.gz -C /opt \
        && cd /opt/mha4mysql-node \
        && perl Makefile.PL \
        && make \
        && make install \
        && tar -zxf /tmp/mha4mysql-manager.tar.gz -C /opt \
        && cd /opt/mha4mysql-manager \
        && perl Makefile.PL \
        && make \
        && make install \
        && cd /opt \
        && rm -rf /opt/mha4mysql-* \
        && apt-get clean

（建议）node和manager一体的Dockerfile内容如下：

    FROM mysql:5.6

    COPY ./mha4mysql-manager-0.55.tar.gz /tmp/
    COPY ./mha4mysql-node-0.54.tar.gz /tmp/
    COPY ./purge_relay_log.sh /home/

    RUN build_deps='ssh sshpass perl libdbi-perl libmodule-install-perl libdbd-mysql-perl libconfig-tiny-perl liblog-dispatch-perl libparallel-forkmanager-perl make cron vim net-tools' \
        && apt-get update \
        && apt-get -y --force-yes install $build_deps \
        && tar -zxf /tmp/mha4mysql-node-0.54.tar.gz -C /opt \
        && cd /opt/mha4mysql-node-0.54 \
        && perl Makefile.PL \
        && make \
        && make install \
        && tar -zxf /tmp/mha4mysql-manager-0.55.tar.gz -C /opt \
        && cd /opt/mha4mysql-manager-0.55 \
        && perl Makefile.PL \
        && make \
        && make install \
        && cd /opt \
        && rm -rf /opt/mha4mysql-* \
        && apt-get clean \
        && ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N '' -y \
        && ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N '' -y \
        && ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N '' -y \
        && sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config \
        && sed -i 's/#   StrictHostKeyChecking ask/StrictHostKeyChecking no/' /etc/ssh/ssh_config \
        && sed -i 's/GSSAPIAuthentication yes/GSSAPIAuthentication no/' /etc/ssh/ssh_config \
        && echo "0 4 * * * /bin/bash /home/purge_relay_log.sh" >> /var/spool/cron/crontabs/root \
        && /etc/init.d/cron restart \
        && 

构建镜像： `docker build -t mha4mysql-node-manager:1.0 .`

验证docker：

    mkdir conf logs data
    docker run -p 3311:3306 --name mysql-instance -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mha4mysql-node-manager:1.0

## 2.2 搭建主从复制环境
目标环境：

|角色             |ip地址        |主机名(docker请无视)    |server_id |类型|
|--|--|
|Monitor host    |172.18.17.5      |server01 |     -     |监控复制组
|Master          |172.18.17.5:3320 |server02 |     101    |  写入
|Candicate master|172.18.17.5:3321 |server03 |     102    |  读
|Slave           |172.18.17.5:3322 |server04 |     103    |  读

### 2.2.1 主从my.cnf配置

Master：

    $ mkdir master
    $ cd master
    $ mkdir conf logs data
    $ vi conf/my.cnf
    [mysqld]
    server-id=101
    log-bin=mysql-bin
    log-slave-updates=true                      #将复制事件写入binlog,一台服务器既做主库又做从库此选项必须要开启
    gtid-mode=on
    enforce-gtid-consistency=true
    master-info-repository=TABLE                #主服信息记录库=表 /文件
    relay-log-info-repository=TABLE             #中继日志信息记录库
    sync-master-info=1
    sync_binlog=0
    binlog-checksum=CRC32
    master-verify-checksum=1
    slave-sql-verify-checksum=1
    replicate-ignore-db = mysql                 #忽略不同步主从的数据库
    replicate-ignore-db = information_schema
    replicate-ignore-db = performance_schema
    init-connect='SET NAMES utf8'               #连接时执行的SQL
    character-set-server=utf8                   #服务端默认字符集
    
    [mysqldump]
    quick
    max_allowed_packet = 16M
    
    $ docker run -p 3320:3306 --name mysql-master -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mha4mysql-node-manager:1.0

Candicate master（不同处：dir、server-id、docker端口、docker名字）：

    $ mkdir cand-master
    $ cd cand-master
    $ mkdir conf logs data
    $ vi conf/my.cnf
    [mysqld]
    server-id=102
    log-bin=mysql-bin
    log-slave-updates=true                      #将复制事件写入binlog,一台服务器既做主库又做从库此选项必须要开启
    gtid-mode=on
    enforce-gtid-consistency=true
    master-info-repository=TABLE                #主服信息记录库=表 /文件
    relay-log-info-repository=TABLE             #中继日志信息记录库
    sync-master-info=1
    sync_binlog=0
    binlog-checksum=CRC32
    master-verify-checksum=1
    slave-sql-verify-checksum=1
    replicate-ignore-db = mysql                 #忽略不同步主从的数据库
    replicate-ignore-db = information_schema
    replicate-ignore-db = performance_schema
    init-connect='SET NAMES utf8'               #连接时执行的SQL
    character-set-server=utf8                   #服务端默认字符集
    
    [mysqldump]
    quick
    max_allowed_packet = 16M
    
    $ docker run -p 3321:3306 --name mysql-cand-master -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mha4mysql-node-manager:1.0

Slave（不同处：dir、server-id、docker端口、docker名字）：

    $ mkdir slave
    $ cd slave
    $ mkdir conf logs data
    $ vi conf/my.cnf
    [mysqld]
    server-id=103
    log-bin=mysql-bin
    log-slave-updates=true                      #将复制事件写入binlog,一台服务器既做主库又做从库此选项必须要开启
    gtid-mode=on
    enforce-gtid-consistency=true
    master-info-repository=TABLE                #主服信息记录库=表 /文件
    relay-log-info-repository=TABLE             #中继日志信息记录库
    sync-master-info=1
    sync_binlog=0
    binlog-checksum=CRC32
    master-verify-checksum=1
    slave-sql-verify-checksum=1
    replicate-ignore-db = mysql                 #忽略不同步主从的数据库
    replicate-ignore-db = information_schema
    replicate-ignore-db = performance_schema
    init-connect='SET NAMES utf8'               #连接时执行的SQL
    character-set-server=utf8                   #服务端默认字符集
    
    [mysqldump]
    quick
    max_allowed_packet = 16M
    
    $ docker run -p 3322:3306 --name mysql-slave -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mha4mysql-node-manager:1.0

### 2.2.2 数据库配置

（1）在master上执行备份

    $ mysqldump -h172.18.17.5 -P3320 -uroot -p123456 --master-data=2 --single-transaction -R --triggers -A > all.sql

其中--master-data=2代表备份时刻记录master的Binlog位置和Position，--single-transaction意思是获取一致性快照，-R意思是备份存储过程和函数，--triggres的意思是备份触发器，-A代表备份所有的库。更多信息请自行mysqldump --help查看。

（2）在master上创建复制用户：

    $ mysql -h172.18.17.5 -P3320 -uroot -p123456
    mysql> grant replication slave on *.* to 'repl'@'%' identified by '123456';
    mysql> flush privileges;

在master上创建监控用户：

    mysql> grant all privileges on *.* to 'root'@'%' identified  by '123456';
    mysql> flush  privileges;

（3）查看主库备份时的binlog名称和位置，MASTER_LOG_FILE和MASTER_LOG_POS：

    $ head -n 30 all.sql | grep 'CHANGE MASTER TO'
    -- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000004', MASTER_LOG_POS=191;

（4）把备份复制到cand-master并导入备份，执行复制相关命令

    $ mysql -h172.18.17.5 -P3321 -uroot -p123456 -e 'reset master;'
    $ mysql -h172.18.17.5 -P3321 -uroot -p123456 < all.sql
    $ mysql -h172.18.17.5 -P3321 -uroot -p123456
    mysql> CHANGE MASTER TO MASTER_HOST='172.18.17.5',
        MASTER_PORT=3320,
        MASTER_USER='repl', 
        MASTER_PASSWORD='123456',
        MASTER_LOG_FILE='mysql-bin.000004',
        MASTER_LOG_POS=191;
    Query OK, 0 rows affected (0.02 sec)

    mysql> start slave;
    Query OK, 0 rows affected (0.01 sec)

在cand-master上查看复制状态（可以看见复制成功）：

    $ mysql -h172.18.17.5 -P3321 -uroot -p123456 -e 'show slave status\G' | egrep 'Slave_IO|Slave_SQL'
        Slave_IO_State: Waiting for master to send event
        Slave_IO_Running: Yes
        Slave_SQL_Running: Yes

（5）把备份复制到slave并导入备份，执行复制相关命令。（和上面一样）

    $ mysql -h172.18.17.5 -P3322 -uroot -p123456 -e 'reset master;'
    $ mysql -h172.18.17.5 -P3322 -uroot -p123456 < all.sql
    $ mysql -h172.18.17.5 -P3322 -uroot -p123456
    mysql> CHANGE MASTER TO MASTER_HOST='172.18.17.5',
        MASTER_PORT=3320,
        MASTER_USER='repl', 
        MASTER_PASSWORD='123456',
        MASTER_LOG_FILE='mysql-bin.000004',
        MASTER_LOG_POS=191;
    Query OK, 0 rows affected (0.02 sec)

    mysql> start slave;
    Query OK, 0 rows affected (0.01 sec)

在slave上查看复制状态（可以看见复制成功）：

    $ mysql -h172.18.17.5 -P3322 -uroot -p123456 -e 'show slave status\G' | egrep 'Slave_IO|Slave_SQL'
        Slave_IO_State: Waiting for master to send event
        Slave_IO_Running: Yes
        Slave_SQL_Running: Yes

（6）两台slave服务器设置read_only（从库对外提供读服务，只所以没有写进配置文件，是因为随时slave会提升为master）

    $ mysql -h172.18.17.5 -P3321 -uroot -p123456 -e 'set global read_only=1'
    $ mysql -h172.18.17.5 -P3322 -uroot -p123456 -e 'set global read_only=1'

到这里整个集群环境已经搭建完毕，剩下的就是配置MHA软件了。

### 2.2.3 配置MHA
(0) 启动manager镜像

    $ docker run --name mysql-manager -it -e MYSQL_ROOT_PASSWORD=123456 mha4mysql-node-manager:1.0 bash
    $ mkdir /etc/masterha

(1）在manager上创建MHA的工作目录，并且创建相关配置文件（在软件包解压后的目录里面有样例配置文件）。

创建app1.cnf配置文件，修改后的文件内容如下（注意，配置文件中的注释可去掉）：

    $ cat app1.cnf 

    [server default]
    manager_workdir=/var/log/masterha/app1.log              #设置manager的工作目录
    manager_log=/var/log/masterha/app1/manager.log          #设置manager的日志
    master_binlog_dir=/data/mysql                           #设置master #保存binlog的位置，以便MHA可以找到master的日志，我这里的也就是mysql的数据目录
    master_ip_failover_script= /usr/local/bin/master_ip_failover    #设置自动failover时候的切换脚本
    master_ip_online_change_script= /usr/local/bin/master_ip_online_change  #设置手动切换时候的切换脚本
    password=123456         #设置mysql中root用户的密码，这个密码是前文中创建监控用户的那个密码
    user=root               #设置监控用户root
    ping_interval=1         #设置监控主库，发送ping包的时间间隔，默认是3秒，尝试三次没有回应的时候自动进行railover
    remote_workdir=/tmp     #设置远端mysql在发生切换时binlog的保存位置
    repl_password=123456    #设置复制用户的密码
    repl_user=repl          #设置复制环境中的复制用户名
    report_script=/usr/local/send_report    #设置发生切换后发送的报警的脚本
    secondary_check_script=/usr/local/bin/masterha_secondary_check -s server03 -s server02            
    shutdown_script=""      #设置故障发生后关闭故障主机脚本（该脚本的主要作用是关闭主机放在发生脑裂,这里没有使用）
    ssh_user=root           #设置ssh的登录用户名

    [server1]
    hostname=172.18.17.5
    port=3320

    [server2]
    hostname=172.18.17.5
    port=3321
    candidate_master=1   #设置为候选master，如果设置该参数以后，发生主从切换以后将会将此从库提升为主库，即使这个主库不是集群中事件最新的slave
    check_repl_delay=0   #默认情况下如果一个slave落后master 100M的relay logs的话，MHA将不会选择该slave作为一个新的master，因为对于这个slave的恢复需要花费很长时间，通过设置check_repl_delay=0,MHA触发切换在选择一个新的master的时候将会忽略复制延时，这个参数对于设置了candidate_master=1的主机非常有用，因为这个候选主在切换的过程中一定是新的master

    [server3]
    hostname=172.18.17.5
    port=3322

纯净版：

    [server default]
    manager_workdir=/var/log/masterha/app1.log 
    manager_log=/var/log/masterha/app1/manager.log
    master_binlog_dir=/data/mysql
    master_ip_failover_script= /usr/local/bin/master_ip_failover
    master_ip_online_change_script= /usr/local/bin/master_ip_online_change
    password=123456
    user=root
    ping_interval=1
    remote_workdir=/tmp
    repl_password=123456
    repl_user=repl
    report_script=/usr/local/send_report
    secondary_check_script=/usr/local/bin/masterha_secondary_check -s 172.18.17.5 -s 172.18.17.5            
    shutdown_script=""
    ssh_user=root

    [server1]
    hostname=172.18.17.5
    port=3320

    [server2]
    hostname=172.18.17.5
    port=3321
    candidate_master=1
    check_repl_delay=0

    [server3]
    hostname=172.18.17.5
    port=3322
    
    $ docker cp ./app1.cnf mysql-manager:/etc/masterha/


（2）设置relay log的清除方式（在每个slave节点上）：

    $ mysql -h172.18.17.5 -P3321 -uroot -p123456 -e 'set global relay_log_purge=0'
    $ mysql -h172.18.17.5 -P3322 -uroot -p123456 -e 'set global relay_log_purge=0'

注意：

MHA在发生切换的过程中，从库的恢复过程中依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF，采用手动清除relay log的方式。在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。但是在MHA环境中，这些中继日志在恢复其他从服务器时可能会被用到，因此需要禁用中继日志的自动删除功能。

定期清除中继日志需要考虑到复制延时的问题。在ext3的文件系统下，删除大的文件需要一定的时间，会导致严重的复制延时。为了避免复制延时，需要暂时为中继日志创建硬链接，因为在linux系统中通过硬链接删除大文件速度会很快。（在mysql数据库中，删除大表时，通常也采用建立硬链接的方式）

MHA节点中包含了pure_relay_logs命令工具，它可以为中继日志创建硬链接，执行SET GLOBAL relay_log_purge=1,等待几秒钟以便SQL线程切换到新的中继日志，再执行SET GLOBAL relay_log_purge=0。

pure_relay_logs脚本参数如下所示：

    --user mysql                      用户名
    --password mysql                  密码
    --port                            端口号
    --workdir                         指定创建relay log的硬链接的位置，默认是/var/tmp，由于系统不同分区创建硬链接文件会失败，故需要执行硬链接具体位置，成功执行脚本后，硬链接的中继日志文件被删除
    --disable_relay_log_purge         默认情况下，如果relay_log_purge=1，脚本会什么都不清理，自动退出，通过设定这个参数，当relay_log_purge=1的情况下会将relay_log_purge设置为0。清理relay log之后，最后将参数设置为OFF。

（3）设置定期清理relay脚本purge_relay_log.sh（两台slave服务器， 注意，这个脚本已经在上文的docker中添加过了）
 
添加到crontab定期执行(这个脚本已经在上文添加过了)

    $ crontab -l
    0 4 * * * /bin/bash /home/purge_relay_log.sh

purge_relay_logs脚本删除中继日志不会阻塞SQL线程。下面我们手动执行看看什么情况。

    $ purge_relay_logs --user=root --password=123456 --port=3306 -disable_relay_log_purge --workdir=/data/
        2014-04-20 15:47:24: purge_relay_logs script started.
        Found relay_log.info: /data/mysql/relay-log.info
        Removing hard linked relay log files server03-relay-bin* under /data/.. done.
        Current relay log file: /data/mysql/server03-relay-bin.000002
        Archiving unused relay log files (up to /data/mysql/server03-relay-bin.000001) ...
        Creating hard link for /data/mysql/server03-relay-bin.000001 under /data//server03-relay-bin.000001 .. ok.
        Creating hard links for unused relay log files completed.
        Executing SET GLOBAL relay_log_purge=1; FLUSH LOGS; sleeping a few seconds so that SQL thread can delete older relay log files (if it keeps up); SET GLOBAL relay_log_purge=0; .. ok.
        Removing hard linked relay log files server03-relay-bin* under /data/.. done.
        2014-04-20 15:47:27: All relay log purging operations succeeded.

### 2.2.4 检查SSH配置

检查MHA Manger到所有MHA Node的SSH连接状态：

    $ docker exec -it mysql-manager bash
    $ masterha_check_ssh --conf=/etc/masterha/app1.cnf 
        Sun Apr 20 17:17:39 2014 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
        Sun Apr 20 17:17:39 2014 - [info] Reading application default configurations from /etc/masterha/app1.cnf..
        Sun Apr 20 17:17:39 2014 - [info] Reading server configurations from /etc/masterha/app1.cnf..
        Sun Apr 20 17:17:39 2014 - [info] Starting SSH connection tests..
        Sun Apr 20 17:17:40 2014 - [debug] 
        Sun Apr 20 17:17:39 2014 - [debug]  Connecting via SSH from root@192.168.0.50(192.168.0.50:22) to root@192.168.0.60(192.168.0.60:22)..
        Sun Apr 20 17:17:39 2014 - [debug]   ok.
        Sun Apr 20 17:17:39 2014 - [debug]  Connecting via SSH from root@192.168.0.50(192.168.0.50:22) to root@192.168.0.70(192.168.0.70:22)..
        Sun Apr 20 17:17:39 2014 - [debug]   ok.
        Sun Apr 20 17:17:40 2014 - [debug] 
        Sun Apr 20 17:17:40 2014 - [debug]  Connecting via SSH from root@192.168.0.60(192.168.0.60:22) to root@192.168.0.50(192.168.0.50:22)..
        Sun Apr 20 17:17:40 2014 - [debug]   ok.
        Sun Apr 20 17:17:40 2014 - [debug]  Connecting via SSH from root@192.168.0.60(192.168.0.60:22) to root@192.168.0.70(192.168.0.70:22)..
        Sun Apr 20 17:17:40 2014 - [debug]   ok.
        Sun Apr 20 17:17:41 2014 - [debug] 
        Sun Apr 20 17:17:40 2014 - [debug]  Connecting via SSH from root@192.168.0.70(192.168.0.70:22) to root@192.168.0.50(192.168.0.50:22)..
        Sun Apr 20 17:17:40 2014 - [debug]   ok.
        Sun Apr 20 17:17:40 2014 - [debug]  Connecting via SSH from root@192.168.0.70(192.168.0.70:22) to root@192.168.0.60(192.168.0.60:22)..
        Sun Apr 20 17:17:41 2014 - [debug]   ok.
        Sun Apr 20 17:17:41 2014 - [info] All SSH connection tests passed successfully.

可以看见各个节点ssh验证都是ok的。

### 2.2.5 检查整个复制环境状况。

(在Manger上）通过masterha_check_repl脚本查看整个集群的状态

    $ masterha_check_repl --conf=/etc/masterha/app1.cnf
        Sun Apr 20 18:36:55 2014 - [info] Checking replication health on 192.168.0.60..
        Sun Apr 20 18:36:55 2014 - [info]  ok.
        Sun Apr 20 18:36:55 2014 - [info] Checking replication health on 192.168.0.70..
        Sun Apr 20 18:36:55 2014 - [info]  ok.
        Sun Apr 20 18:36:55 2014 - [info] Checking master_ip_failover_script status:
        Sun Apr 20 18:36:55 2014 - [info]   /usr/local/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=192.168.0.50 --orig_master_ip=192.168.0.50 --orig_master_port=3306 
        Bareword "FIXME_xxx" not allowed while "strict subs" in use at /usr/local/bin/master_ip_failover line 88.
        Execution of /usr/local/bin/master_ip_failover aborted due to compilation errors.
        Sun Apr 20 18:36:55 2014 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln214]  Failed to get master_ip_failover_script status with return code 255:0.
        Sun Apr 20 18:36:55 2014 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln383] Error happend on checking configurations.  at /usr/local/bin/masterha_check_repl line 48
        Sun Apr 20 18:36:55 2014 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln478] Error happened on monitoring servers.
        Sun Apr 20 18:36:55 2014 - [info] Got exit code 1 (Not master dead).

        MySQL Replication Health is NOT OK!

发现最后的结论说我的复制不是ok的。但是上面的信息明明说是正常的，自己也进数据库查看了。这里一直踩坑。一直纠结，后来无意中发现火丁笔记的博客，这才知道了原因，原来Failover两种方式：一种是虚拟IP地址，一种是全局配置文件。MHA并没有限定使用哪一种方式，而是让用户自己选择，虚拟IP地址的方式会牵扯到其它的软件,比如keepalive软件，而且还要修改脚本master_ip_failover。(最后修改脚本后才没有这个报错，自己不懂perl也是折腾的半死，去年买了块表)

如果发现如下错误：

    Can't exec "mysqlbinlog": No such file or directory at /usr/local/share/perl5/MHA/BinlogManager.pm line 99.
    mysqlbinlog version not found!
    Testing mysql connection and privileges..sh: mysql: command not found

解决方法如下，添加软连接（所有节点）

    ln -s /usr/local/mysql/bin/mysqlbinlog /usr/local/bin/mysqlbinlog
    ln -s /usr/local/mysql/bin/mysql /usr/local/bin/mysql

所以先暂时注释master_ip_failover_script= /usr/local/bin/master_ip_failover这个选项。后面引入keepalived后和修改该脚本以后再开启该选项。

    $ grep master_ip_failover /etc/masterha/app1.cnf
        #master_ip_failover_script= /usr/local/bin/master_ip_failover

再次进行状态查看：

    MySQL Replication Health is OK.

已经没有明显报错，只有两个警告而已，复制也显示正常了。

### 2.2.6 检查MHA Manager的状态：

(在Manger上）通过master_check_status脚本查看Manager的状态：
    
    $ masterha_check_status --conf=/etc/masterha/app1.cnf
        app1 is stopped(2:NOT_RUNNING).

注意：如果正常，会显示"PING_OK"，否则会显示"NOT_RUNNING"，这代表MHA监控没有开启。

### 2.2.7 开启MHA Manager监控

    $ nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/masterha/app1/manager.log 2>&1 &  
        [1] 30867

启动参数介绍：

    --remove_dead_master_conf      该参数代表当发生主从切换后，老的主库的ip将会从配置文件中移除。
    --manger_log                   日志存放位置
    --ignore_last_failover         在缺省情况下，如果MHA检测到连续发生宕机，且两次宕机间隔不足8小时的话，则不会进行Failover，之所以这样限制是为了避免ping-pong效应。该参数代表忽略上次MHA触发切换产生的文件，默认情况下，MHA发生切换后会在日志目录，也就是上面我设置的/data产生app1.failover.complete文件，下次再次切换的时候如果发现该目录下存在该文件将不允许触发切换，除非在第一次切换后收到删除该文件，为了方便，这里设置为--ignore_last_failover。

查看MHA Manager监控是否正常：

    $ masterha_check_status --conf=/etc/masterha/app1.cnf
        app1 (pid:20386) is running(0:PING_OK), master:192.168.0.50

可以看见已经在监控了

### 2.2.8 查看启动日志

    $ tail -n20 /var/log/masterha/app1/manager.log
    Sun Apr 20 19:12:01 2014 - [info]   Connecting to root@192.168.0.70(192.168.0.70:22).. 
      Checking slave recovery environment settings..
        Opening /data/mysql/relay-log.info ... ok.
        Relay log found at /data/mysql, up to server04-relay-bin.000002
        Temporary relay log file is /data/mysql/server04-relay-bin.000002
        Testing mysql connection and privileges.. done.
        Testing mysqlbinlog output.. done.
        Cleaning up test file(s).. done.
    Sun Apr 20 19:12:01 2014 - [info] Slaves settings check done.
    Sun Apr 20 19:12:01 2014 - [info] 
    192.168.0.50 (current master)
     +--192.168.0.60
     +--192.168.0.70

    Sun Apr 20 19:12:01 2014 - [warning] master_ip_failover_script is not defined.
    Sun Apr 20 19:12:01 2014 - [warning] shutdown_script is not defined.
    Sun Apr 20 19:12:01 2014 - [info] Set master ping interval 1 seconds.
    Sun Apr 20 19:12:01 2014 - [info] Set secondary check script: /usr/local/bin/masterha_secondary_check -s server03 -s server02 --user=root --master_host=server02 --master_ip=192.168.0.50 --master_port=3306
    Sun Apr 20 19:12:01 2014 - [info] Starting ping health check on 192.168.0.50(192.168.0.50:3306)..
    Sun Apr 20 19:12:01 2014 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..
 
其中"Ping(SELECT) succeeded, waiting until MySQL doesn't respond.."说明整个系统已经开始监控了。

### 2.2.9 关闭MHA Manager监控

关闭很简单，使用masterha_stop命令完成。

    $ masterha_stop --conf=/etc/masterha/app1.cnf
        Stopped app1 successfully.
        [1]+  Exit 1                  nohup masterha_manager --conf=/etc/masterha/app1.cnf --remove_dead_master_conf --ignore_last_failover --manager_log=/data/mamanager.log

### 2.2.10 配置VIP

vip配置可以采用两种方式，一种通过keepalived的方式管理虚拟ip的浮动；另外一种通过脚本方式启动虚拟ip的方式（即不需要keepalived或者heartbeat类似的软件）。

1.keepalived方式管理虚拟ip，keepalived配置方法如下：

（1）下载软件进行并进行安装（两台master，准确的说一台是master，另外一台是备选master，在没有切换以前是slave）：

    $ wget http://www.keepalived.org/software/keepalived-1.2.12.tar.gz
    $ tar xf keepalived-1.2.12.tar.gz           
    $ cd keepalived-1.2.12
    $ ./configure --prefix=/usr/local/keepalived
    $ make &&  make install
    $ cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
    $ cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
    mkdir /etc/keepalived
    $ cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
    $ cp /usr/local/keepalived/sbin/keepalived /usr/sbin/

（2）配置keepalived的配置文件，在master上配置

    $  cat /etc/keepalived/keepalived.conf
        ! Configuration File for keepalived

        global_defs {
             notification_email {
             saltstack@163.com
           }
           notification_email_from dba@dbserver.com
           smtp_server 127.0.0.1
           smtp_connect_timeout 30
           router_id MySQL-HA
        }

        vrrp_instance VI_1 {
            state BACKUP
            interface eth1
            virtual_router_id 51
            priority 150
            advert_int 1
            nopreempt

            authentication {
            auth_type PASS
            auth_pass 1111
            }

            virtual_ipaddress {
                192.168.0.88
            }
        }

其中router_id MySQL HA表示设定keepalived组的名称，将192.168.0.88这个虚拟ip绑定到该主机的eth1网卡上，并且设置了状态为backup模式，将keepalived的模式设置为非抢占模式（nopreempt），priority 150表示设置的优先级为150。下面的配置略有不同，但是都是一个意思。

在候选master上配置（192.168.0.60）

    $ cat /etc/keepalived/keepalived.conf 
        ! Configuration File for keepalived

        global_defs {
             notification_email {
             saltstack@163.com
           }
           notification_email_from dba@dbserver.com
           smtp_server 127.0.0.1
           smtp_connect_timeout 30
           router_id MySQL-HA
        }

        vrrp_instance VI_1 {
            state BACKUP
            interface eth1
            virtual_router_id 51
            priority 120
            advert_int 1
            nopreempt

            authentication {
            auth_type PASS
            auth_pass 1111
            }

            virtual_ipaddress {
                192.168.0.88
            }
        }

（3）启动keepalived服务，在master上启动并查看日志

    $ /etc/init.d/keepalived start
        Starting keepalived:                                       [  OK  ]
    $ tail -f /var/log/messages
        Apr 20 20:22:16 192 Keepalived_healthcheckers[15334]: Opening file '/etc/keepalived/keepalived.conf'.
        Apr 20 20:22:16 192 Keepalived_healthcheckers[15334]: Configuration is using : 7231 Bytes
        Apr 20 20:22:16 192 kernel: IPVS: Connection hash table configured (size=4096, memory=64Kbytes)
        Apr 20 20:22:16 192 kernel: IPVS: ipvs loaded.
        Apr 20 20:22:16 192 Keepalived_healthcheckers[15334]: Using LinkWatch kernel netlink reflector...
        Apr 20 20:22:19 192 Keepalived_vrrp[15335]: VRRP_Instance(VI_1) Transition to MASTER STATE
        Apr 20 20:22:20 192 Keepalived_vrrp[15335]: VRRP_Instance(VI_1) Entering MASTER STATE
        Apr 20 20:22:20 192 Keepalived_vrrp[15335]: VRRP_Instance(VI_1) setting protocol VIPs.
        Apr 20 20:22:20 192 Keepalived_vrrp[15335]: VRRP_Instance(VI_1) Sending gratuitous ARPs on eth1 for 192.168.0.88
        Apr 20 20:22:20 192 Keepalived_healthcheckers[15334]: Netlink reflector reports IP 192.168.0.88 added
        Apr 20 20:22:25 192 Keepalived_vrrp[15335]: VRRP_Instance(VI_1) Sending gratuitous ARPs on eth1 for 192.168.0.88

发现已经将虚拟ip 192.168.0.88绑定了网卡eth1上。
（4）查看绑定情况

    $ ip addr | grep eth1
        3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
            inet 192.168.0.50/24 brd 192.168.0.255 scope global eth1
            inet 192.168.0.88/32 scope global eth1

在另外一台服务器，候选master上启动keepalived服务，并观察

    $ /etc/init.d/keepalived start ; tail -f /var/log/messages
        Starting keepalived:                                       [  OK  ]
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: Registering gratuitous ARP shared channel
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: Opening file '/etc/keepalived/keepalived.conf'.
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: Configuration is using : 62976 Bytes
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: Using LinkWatch kernel netlink reflector...
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: VRRP_Instance(VI_1) Entering BACKUP STATE
        Apr 20 20:26:18 192 Keepalived_vrrp[9472]: VRRP sockpool: [ifindex(3), proto(112), unicast(0), fd(10,11)]
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Netlink reflector reports IP 192.168.80.138 added
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Netlink reflector reports IP 192.168.0.60 added
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Netlink reflector reports IP fe80::20c:29ff:fe9d:6a9e added
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Netlink reflector reports IP fe80::20c:29ff:fe9d:6aa8 added
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Registering Kernel netlink reflector
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Registering Kernel netlink command channel
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Opening file '/etc/keepalived/keepalived.conf'.
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Configuration is using : 7231 Bytes
        Apr 20 20:26:18 192 kernel: IPVS: Registered protocols (TCP, UDP, AH, ESP)
        Apr 20 20:26:18 192 kernel: IPVS: Connection hash table configured (size=4096, memory=64Kbytes)
        Apr 20 20:26:18 192 kernel: IPVS: ipvs loaded.
        Apr 20 20:26:18 192 Keepalived_healthcheckers[9471]: Using LinkWatch kernel netlink reflector...

从上面的信息可以看到keepalived已经配置成功。

注意：

上面两台服务器的keepalived都设置为了BACKUP模式，在keepalived中2种模式，分别是master->backup模式和backup->backup模式。这两种模式有很大区别。

在master->backup模式下，一旦主库宕机，虚拟ip会自动漂移到从库，当主库修复后，keepalived启动后，还会把虚拟ip抢占过来，即使设置了非抢占模式（nopreempt）抢占ip的动作也会发生。

在backup->backup模式下，当主库宕机后虚拟ip会自动漂移到从库上，当原主库恢复和keepalived服务启动后，并不会抢占新主的虚拟ip，即使是优先级高于从库的优先级别，也不会发生抢占。为了减少ip漂移次数，通常是把修复好的主库当做新的备库。

（5）MHA引入keepalived（MySQL服务进程挂掉时通过MHA 停止keepalived）:

要想把keepalived服务引入MHA，我们只需要修改切换是触发的脚本文件master_ip_failover即可，在该脚本中添加在master发生宕机时对keepalived的处理。

编辑脚本/usr/local/bin/master_ip_failover，修改后如下，（主库上操作，192.168.0.50）。

在MHA Manager修改脚本修改后的内容如下（参考资料比较少）：

    #!/usr/bin/env perl

    use strict;
    use warnings FATAL => 'all';

    use Getopt::Long;

    my (
        $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
        $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
    );

    my $vip = '192.168.0.88';
    my $ssh_start_vip = "/etc/init.d/keepalived start";
    my $ssh_stop_vip = "/etc/init.d/keepalived stop";

    GetOptions(
        'command=s'          => \$command,
        'ssh_user=s'         => \$ssh_user,
        'orig_master_host=s' => \$orig_master_host,
        'orig_master_ip=s'   => \$orig_master_ip,
        'orig_master_port=i' => \$orig_master_port,
        'new_master_host=s'  => \$new_master_host,
        'new_master_ip=s'    => \$new_master_ip,
        'new_master_port=i'  => \$new_master_port,
    );

    exit &main();

    sub main {

        print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

        if ( $command eq "stop" || $command eq "stopssh" ) {

            my $exit_code = 1;
            eval {
                print "Disabling the VIP on old master: $orig_master_host \n";
                &stop_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn "Got Error: $@\n";
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "start" ) {

            my $exit_code = 10;
            eval {
                print "Enabling the VIP - $vip on the new master - $new_master_host \n";
                &start_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn $@;
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "status" ) {
            print "Checking the Status of the script.. OK \n";
            #`ssh $ssh_user\@cluster1 \" $ssh_start_vip \"`;
            exit 0;
        }
        else {
            &usage();
            exit 1;
        }
    }

    # A simple system call that enable the VIP on the new master
    sub start_vip() {
        `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    }
    # A simple system call that disable the VIP on the old_master
    sub stop_vip() {
         return 0  unless  ($ssh_user);
        `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
    }

    sub usage {
        print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
    }

现在已经修改这个脚本了，我们现在打开在上面提到过的参数，再检查集群状态，看是否会报错。

    $ grep 'master_ip_failover_script' /etc/masterha/app1.cnf
        master_ip_failover_script= /usr/local/bin/master_ip_failover
    $ masterha_check_repl --conf=/etc/masterha/app1.cnf  
        Sun Apr 20 23:10:01 2014 - [info] Slaves settings check done.
        Sun Apr 20 23:10:01 2014 - [info] 
        192.168.0.50 (current master)
         +--192.168.0.60
         +--192.168.0.70

        Sun Apr 20 23:10:01 2014 - [info] Checking replication health on 192.168.0.60..
        Sun Apr 20 23:10:01 2014 - [info]  ok.
        Sun Apr 20 23:10:01 2014 - [info] Checking replication health on 192.168.0.70..
        Sun Apr 20 23:10:01 2014 - [info]  ok.
        Sun Apr 20 23:10:01 2014 - [info] Checking master_ip_failover_script status:
        Sun Apr 20 23:10:01 2014 - [info]   /usr/local/bin/master_ip_failover --command=status --ssh_user=root --orig_master_host=192.168.0.50 --orig_master_ip=192.168.0.50 --orig_master_port=3306 
        Sun Apr 20 23:10:01 2014 - [info]  OK.
        Sun Apr 20 23:10:01 2014 - [warning] shutdown_script is not defined.
        Sun Apr 20 23:10:01 2014 - [info] Got exit code 0 (Not master dead).

        MySQL Replication Health is OK.

可以看见已经没有报错了。

 /usr/local/bin/master_ip_failover添加或者修改的内容意思是当主库数据库发生故障时，会触发MHA切换，MHA Manager会停掉主库上的keepalived服务，触发虚拟ip漂移到备选从库，从而完成切换。当然可以在keepalived里面引入脚本，这个脚本监控mysql是否正常运行，如果不正常，则调用该脚本杀掉keepalived进程。

2.通过脚本的方式管理VIP。

这里是修改/usr/local/bin/master_ip_failover，也可以使用其他的语言完成，比如php语言。使用php脚本编写的failover这里就不介绍了。修改完成后内容如下，而且如果使用脚本管理vip的话，需要手动在master服务器上绑定一个vip

    $ /sbin/ifconfig eth1:1 192.168.0.88/24

通过脚本来维护vip的测试我这里就不说明了，童鞋们自行测试，脚本如下（测试通过）

    #!/usr/bin/env perl

    use strict;
    use warnings FATAL => 'all';

    use Getopt::Long;

    my (
        $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
        $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
    );

    my $vip = '192.168.0.88/24';
    my $key = '1';
    my $ssh_start_vip = "/sbin/ifconfig eth1:$key $vip";
    my $ssh_stop_vip = "/sbin/ifconfig eth1:$key down";

    GetOptions(
        'command=s'          => \$command,
        'ssh_user=s'         => \$ssh_user,
        'orig_master_host=s' => \$orig_master_host,
        'orig_master_ip=s'   => \$orig_master_ip,
        'orig_master_port=i' => \$orig_master_port,
        'new_master_host=s'  => \$new_master_host,
        'new_master_ip=s'    => \$new_master_ip,
        'new_master_port=i'  => \$new_master_port,
    );

    exit &main();

    sub main {

        print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";

        if ( $command eq "stop" || $command eq "stopssh" ) {

            my $exit_code = 1;
            eval {
                print "Disabling the VIP on old master: $orig_master_host \n";
                &stop_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn "Got Error: $@\n";
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "start" ) {

            my $exit_code = 10;
            eval {
                print "Enabling the VIP - $vip on the new master - $new_master_host \n";
                &start_vip();
                $exit_code = 0;
            };
            if ($@) {
                warn $@;
                exit $exit_code;
            }
            exit $exit_code;
        }
        elsif ( $command eq "status" ) {
            print "Checking the Status of the script.. OK \n";
            exit 0;
        }
        else {
            &usage();
            exit 1;
        }
    }

    sub start_vip() {
        `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    }
    sub stop_vip() {
         return 0  unless  ($ssh_user);
        `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
    }

    sub usage {
        print
        "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
    }

为了防止脑裂发生，推荐生产环境采用脚本的方式来管理虚拟ip，而不是使用keepalived来完成。到此为止，基本MHA集群已经配置完毕。接下来就是实际的测试环节了。通过一些测试来看一下MHA到底是如何进行工作的。下面将从MHA自动failover，我们手动failover，在线切换三种方式来介绍MHA的工作情况。

## 三、 恢复

### 3.1 恢复办法

1. master的mysqld被kill掉：mha会自动切换master，要做的工作是在切换后把master上的mysqld重启。
2. master整个宕机：mha已经不能连接master获取slave信息，此时要手动切换haproxy，将连接导向slave。再进行后续恢复。

### 3.2 master容器宕机

重启master容器后，发现ssh走不通了！

    SSH connection from root@master(192.168.0.2:22) to root@slave_1(192.168.0.3:22) failed!

运行repair_ssh.sh脚本修复，再运行monitor.sh

    ./repair_ssh.sh
    ./monitor.sh

## REF
> [mysql-master-ha](https://code.google.com/p/mysql-master-ha/)  
> [MySQL高可用架构之MHA 原理与实践](https://www.cnblogs.com/rayment/p/7355093.html)  
> [基于Docker的mysql mha 的集群环境构建实践](https://dbaplus.cn/news-11-129-1.html)  
> [（docker+mha) linux 批量ssh互信创建 自动化脚本](https://blog.csdn.net/u010719917/article/details/86765872)
