---
layout: post
title: 高可用MySQL方案
category: mysql
comments: false
---
## 一、一些术语

MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。

Galera Cluster是MariaDB的一个双活多主集群，当前最新版本10.0.30，其可以使得MariDB的所有节点保持同步，Galera为MariaDB提供了同步复制（相对于原生的异步复制），因此其可以保证HA，且其当前仅支持XtraDB/InnoDB存储引擎（扩展支持MyISAM），并且只可在Linux下使用。

MMM （Multi-Master Replicatin Manager）是用perl语言开发的一套用于管理mysql主主同步架构的一种工具集，主要作用是监控和管理mysql的主主复制拓扑，并在当前的主服务器失效时，进行主和主备服务器之间的主从切换和故障转移工作。

MHA（Master High Availability）目前在MySQL高可用方面是一个相对成熟的解决方案，是一套优秀的作为MySQL高可用性环境下故障切换和主从提升的高可用软件。在MySQL故障切换过程中，MHA能做到在0`~`30秒之内自动完成数据库的故障切换操作，并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的高可用。

GTID即全局事务ID（global transaction identifier），GTID实际上是由UUID+TID组成的。其中UUID是一个MySQL实例的唯一标识。TID代表了该实例上已经提交的事务数量，并且随着事务提交单调递增，所以GTID能够保证每个MySQL实例事务的执行（不会重复执行同一个事务，并且会补全没有执行的事务）。这是MySQL5.6在5.5的基础上增加的一项改进。

## 二、可选MySQL高可用方案
MySQL的各种高可用方案，大多是基于以下几种基础来部署的：

- 基于主从复制；
- 基于Galera协议；
- 基于NDB引擎；
- 基于中间件/proxy；
- 基于共享存储；
- 基于主机高可用；

在这些可选项中，最常见的就是基于主从复制的方案，其次是基于Galera的方案，我们重点说说这两种方案。其余几种方案在生产上用的并不多，我们只简单说下。




## REF
> [MYSQL(高可用方案)](https://www.cnblogs.com/robbinluobo/p/8294782.html)  
> [浅谈MariaDB Galera Cluster架构](https://www.cnblogs.com/vadim/p/6930566.html)  
> [五大常见的MySQL高可用方案](https://www.cnblogs.com/Kellana/p/6738739.html)  
> [MySQL5.6 新特性之GTID](https://www.cnblogs.com/zhoujinyi/p/4717951.html)  
> [Mysql架构MMM,MHA](https://blog.csdn.net/qq_30353203/article/details/78253700)  
> [mariaDB](https://baike.baidu.com/item/mariaDB/6466119?fr=aladdin)