---
layout: post
title: percona toolkit
category: mysql
comments: false
---

percona toolkit 是一款[percona](http://www.percona.com/)公司推出的优秀的开源的mysql分析工具。It can

- 检查master和slave数据的一致性
- 有效地对记录进行归档
- 查找重复的索引
- 对服务器信息进行汇总
- 分析来自日志和tcpdump的查询
- 当系统出问题的时候收集重要的系统信息

## 一、安装

### 1.1 检查和安装与Perl相关的模块

PT工具是使用Perl语言编写和执行的，所以需要系统中有Perl环境。

依赖包检查命令为：

    # rpm -qa perl-DBI perl-DBD-MySQL perl-Time-HiRes perl-IO-Socket-SSL
    # yum -y install perl-DBI
    # yum -y install perl-DBD-MySQL
    # yum -y install perl-Time-HiRes
    # yum -y install perl-IO-Socket-SSL
    # yum -y install perl-Digest-MD5
    # yum -y install perl-TermReadKey

提前安装好MySQL-shared-5.6.43-1.el7.x86_64.rpm：https://cdn.mysql.com//Downloads/MySQL-5.6/MySQL-shared-5.6.43-1.el7.x86_64.rpm

### 1.2 安装pt
下载页面：[https://www.percona.com/downloads/percona-toolkit/LATEST/](https://www.percona.com/downloads/percona-toolkit/LATEST/)

    wget https://www.percona.com/downloads/percona-toolkit/3.0.13/binary/redhat/7/x86_64/percona-toolkit-3.0.13-re85ce15-el7-x86_64-bundle.tar
    tar -xf percona-toolkit-3.0.13-re85ce15-el7-x86_64-bundle.tar
    rpm -iv percona-toolkit-3.0.13-1.el7.x86_64.rpm
    rpm -iv percona-toolkit-debuginfo-3.0.13-1.el7.x86_64.rpm

## 二、使用

### 2.1 使用pt-heartbeat检测主从复制延迟

不要用SECONDS_BEHIND_MASTER来衡量MYSQL主备的延迟时间，原因如下：

A：备库Seconds_behand_master值是通过将服务器当前的时间戳与二进制日志中的事件的时间戳对比得到的，所以只有在执行事件时才能报告延迟

B：如果备库复制线程没有运行，就会报延迟为null

C：一些错误，如主备的max_allowed_packet不匹配或者网络不稳定时，可能中断复制或者停止复制线程，但Seconds_behand_master将显示为0而不是显示错误

D：即使备库线程正在运行，备库有时候可能无法计算延迟时，如果发生这种情况，备库会报0或者null

E：一个较大的事务可能导致延迟波动，如：有一个事务更新数据长达一个小时，最后提交，这条更新将比它实际发生时间要晚一个小时才记录到二进制日志中，当备库执行这条语句时，会临时地报告备库延迟一个小时，然后很快又变回0

F：如果分发主库落后了，并且其本身也有已经追赶上它的备库，备库的延迟将显示为0，而事实上备库和源主库之间此时是有延迟的。

解决这些问题的办法是忽略这个值，并使用一些可以直接观察和衡量的方式来监控备库延迟，最好的解决办法是使用heartbeat record，这是一个在主库上每秒更新一次的时间戳，为了计算延迟，可以直接用备库当前的时间戳减去心跳记录的值，这个方法能够解决刚刚提到的所有问题，另外一个额外的好处是我们还可以通过时间戳知道备库当前的复制状况，包含在percona toolkit里的pt-heartbeat脚本是复制心跳的最流行的一种实现。

心跳还有其他好处，记录在二进制日志中的心跳记录拥有许多用途，如：在一些很难解决的场景下可以用于灾难恢复。

pt-heartbeat的工作原理：

1、在主上创建一张heartbeat表，按照一定的时间频率更新该表的字段（把时间更新进去）。

2、从主库连接到从上检查复制的时间记录，和从库的当前系统时间进行比较，得出时间的差异。

使用方法：

    shell > pt-heartbeat [OPTIONS] [DSN] --update|--monitor|--check|--stop

options可以自行查看help（或加--help查看有哪些选项）,DNS为你要操作的数据库和表。

在主库上创建一个测试库，测试库里创建好heartbeat表，然后开启守护进程来更新felix.heartbeat表：

    mysql> create database felix; use felix;

    mysql> CREATE TABLE heartbeat (
      ts                    varchar(26) NOT NULL,
      server_id             int unsigned NOT NULL PRIMARY KEY,
      file                  varchar(255) DEFAULT NULL,    -- SHOW MASTER STATUS
      position              bigint unsigned DEFAULT NULL, -- SHOW MASTER STATUS
      relay_master_log_file varchar(255) DEFAULT NULL,    -- SHOW SLAVE STATUS
      exec_master_log_pos   bigint unsigned DEFAULT NULL  -- SHOW SLAVE STATUS
    );

    shell> pt-heartbeat -D felix -hlocalhost -P3307 -uroot -p123456 --interval=1 --log=/opt/master-slave.txt --create-table --update --replace --daemonize
    shell> ps -ef|grep pt-heartbeat

在主库上监控从的延迟情况：
    
    shell > pt-heartbeat -D felix --monitor -hlocalhost -uroot -p123456 --master-server-id=102 #一直执行，不退出
    或
    shell > pt-heartbeat -D felix --check -h 192.168.16.103    # 执行一次就退出
    0.00s [ 0.00s, 0.00s, 0.00s ]
    0.00s [ 0.00s, 0.00s, 0.00s ]
    0.00s [ 0.00s, 0.00s, 0.00s ]
    0.00s [ 0.00s, 0.00s, 0.00s ]

0表示从没有延迟。 [ 0.00s, 0.00s, 0.00s ] 表示1m,5m,15m的平均值。可以通过--frames去设置。

--check  
检查从的延迟，检查一次就退出，除非指定了--recurse会递归的检查所有的从服务器。

--check-read-only  
如果从服务器开启了只读模式，该工具会跳过任何插入。

--daemonize  
执行时，放入到后台执行

--interval  
检查、更新的间隔时间。默认是见是1s。最小的单位是0.01s，最大精度为小数点后两位，因此0.015将调整至0.02。

--log  
开启daemonized模式的所有日志将会被打印到制定的文件中。

--monitor  
持续监控从的延迟情况。通过--interval指定的间隔时间，打印出从的延迟信息，通过--file则可以把这些信息打印到指定的文件。

关闭heartbeat:

    shell> pt-heartbeat --stop

后续要继续开启后台进行的话，记住一定要先把/tmp/pt-heartbeat-sentinel 文件删除，否则启动不了。

### 2.2 主从同步延迟监控脚本
定期过滤--log文件中最大值（此脚本运行的前提是：启动更新主库heartbeat命令以及带上--log的同步延迟检测命令）。如果发生延迟，发送报警邮件。

    [root@master-server ~]# cat /root/check-slave-monit.sh 
    #!/bin/bash
    cat /opt/master-slave.txt > /opt/master_slave.txt
    echo > /opt/master-slave.txt
    max_time=`cat /opt/master_slave.txt |grep -v '^$' |awk '{print $1}' |sort -k1nr |head -1`
    NUM=$(echo "$max_time"|cut -d"s" -f1)
    if [ $NUM == "0.00" ];then
       echo "Mysql主从数据一致"
    else
       /usr/local/bin/sendEmail -f ops@huanqiu.cn -t wangshibo@huanqiu.cn -s smtp.huanqiu.cn -u "Mysql主从同步延迟" -o message-content-type=html -o message-charset=utf8 -xu ops@huanqiu.cn -xp WEE78@12l$ -m "Mysql主从数据同步有延迟"
    fi
    [root@master-server ~]# chmod /root/check-slave-monit.sh
    [root@master-server ~]# sh /root/check-slave-monit.sh 
    Mysql主从数据一致

结合crontab，每隔一分钟检查一次

    [root@master-server ~]# crontab -e
    #mysql主从同步延迟检查
    * * * * * /bin/bash -x /root/check-slave-monit.sh > /dev/null 2>&1


sendemail邮件发送环境部署参考：http://www.cnblogs.com/kevingrace/p/5961861.html

### 2.3 pt-slave-find
查找和打印mysql所有从服务器复制层级关系.

    [root@instance-3d7aof0v-1 ~]# pt-slave-find -hlocalhost -uroot -p123456
    Cannot connect to h=host-192-168-16-103.bj,p=...,u=root
    Cannot connect to h=host-192-168-16-104.bj,p=...,u=root
    localhost
    Version         5.6.43-log
    Server ID       102
    Uptime          05:34:16 (started 2019-02-13T12:09:02)
    Replication     Is not a slave, has 2 slaves connected, is not read_only
    Filters
    Binary logging  STATEMENT
    Slave status
    Slave mode      STRICT
    Auto-increment  increment 1, offset 1
    InnoDB version  5.6.43

原理：连接mysql主服务器并查找其所有的从，然后打印出所有从服务器的层级关系。

### 2.4 pt-duplicate-key-checker
为从mysql表中找出重复的索引和外键，这个工具会将重复的索引和外键都列出来，并生成了删除重复索引的语句，非常方便。

    [root@instance-3d7aof0v-1 ~]# pt-duplicate-key-checker --databases=test -hlocalhost -uroot -p123456
    # ########################################################################
    # Summary of indexes
    # ########################################################################

    # Total Indexes  2

### 2.5 pt-mysql-summary
精细地对mysql的配置和sataus信息进行汇总，汇总后你直接看一眼就能看明白。

工作原理：连接mysql后查询出status和配置信息保存到临时目录中，然后用awk和其他的脚本工具进行格式化。OPTIONS可以查阅官网的相关页面。

用法介绍：
pt-mysql-summary [OPTIONS] [-- MYSQL OPTIONS]

汇总本地mysql服务器的status和配置信息：

    pt-mysql-summary -- -hlocalhost -uroot -p123456  

### 2.6 pt-slave-restart
监视mysql复制错误，并尝试重启mysql复制当复制停止的时候

用法介绍：
pt-slave-restart [OPTION...] [DSN]

监视一个或者多个mysql复制错误，当从停止的时候尝试重新启动复制。你可以指定跳过的错误并运行从到指定的日志位置。

使用示例：监视192.168.1.101的从，跳过1个错误

    [root@master-server ~]# pt-slave-restart --user=root --password=123456 --host=192.168.1.101 --skip-count=1

### 2.7 pt-table-checksum
用于检测MySQL主、从库的数据是否一致。

其原理是在主库执行基于statement的sql语句来生成主库数据块的checksum，把相同的sql语句传递到从库执行，并在从库上计算相同数据块的checksum，最后，比较主从库上相同数据块的checksum值，由此判断主从数据是否一致。

检测过程根据唯一索引将表按row切分为块（chunk），以为单位计算，可以避免锁表。检测时会自动判断复制延迟、 master的负载， 超过阀值后会自动将检测暂停，减小对线上服务的影响。



## REF
> [linux下percona-toolkit工具包的安装和使用（超详细版）](https://www.cnblogs.com/zishengY/p/6852280.html)  
> [使用pt-heartbeat检测主从复制延迟](https://www.cnblogs.com/xiaoboluo768/p/5147425.html)  
> [pt-heartbeat监控mysql主从复制延迟整理](https://blog.csdn.net/qq_33285112/article/details/78794075)  
> [生产环境使用 pt-table-checksum 检查MySQL数据一致性](https://segmentfault.com/a/1190000004309169)  
> [mysql主从同步(3)-percona-toolkit工具（数据一致性监测、延迟监控）使用梳理](https://www.cnblogs.com/kevingrace/p/6261091.html)