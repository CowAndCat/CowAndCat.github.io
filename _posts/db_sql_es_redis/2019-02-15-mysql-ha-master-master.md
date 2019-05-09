---
layout: post
title: MySQL双主（主主）架构方案MMM
category: mysql
comments: false
---

在企业中，数据库高可用一直是企业的重中之重，中小企业很多都是使用mysql主从方案，一主多从，读写分离等，但是单主存在单点故障，从库切换成主库需要作改动。因此，如果是双主或者多主，就会增加mysql入口，增加高可用。不过多主需要考虑自增长ID问题，这个需要特别设置配置文件，比如双主，可以使用奇偶，总之，主之间设置自增长ID相互不冲突就能完美解决自增长ID冲突问题。

## 一、MySQL双主（主主）架构
MySQL双主（主主）架构方案思路是:

1.两台mysql都可读写，互为主备，默认只使用一台（masterA）负责数据的写入，另一台（masterB）备用；  
2.masterA是masterB的主库，masterB又是masterA的主库，它们互为主从；  
3.两台主库之间做高可用,可以采用keepalived等方案（使用VIP对外提供服务）；  
4.所有提供服务的从服务器与masterB进行主从同步（双主多从）;  
5.建议采用高可用策略的时候，masterA或masterB均不因宕机恢复后而抢占VIP（非抢占模式）；

这样做可以在一定程度上保证主库的高可用,在一台主库down掉之后,可以在极短的时间内切换到另一台主库上（尽可能减少主库宕机对业务造成的影响），减少了主从同步给线上主库带来的压力；

但是也有几个不足的地方:

1.masterB可能会一直处于空闲状态（可以用它当从库，负责部分查询）；  
2.主库后面提供服务的从库要等masterB先同步完了数据后才能去masterB上去同步数据，这样可能会造成一定程度的同步延时；

不要将read_only=1写进从库的配置文件，因为主库宕机时，从库要提升为主库接受写请求  mysql -e"set global read_only=1"

## REF
> [MySQL双主（主主）架构](https://www.cnblogs.com/ygqygq2/p/6045279.html)  
> [LVS+Keepalived实现mysql的负载均衡](https://www.cnblogs.com/tangyanbo/p/4305589.html)  
> [企业级-Mysql双主互备高可用负载均衡架构（基于GTID主从复制模式）](https://www.cnblogs.com/liangshaoye/p/5459421.html)