---
layout: post
title: Redis与Memcached的区别
category: tech
comments: false
---
##1、传统MySQL+ Memcached架构遇到的问题
实际MySQL是适合进行海量数据存储的，通过Memcached将热点数据加载到cache，加速访问，很多公司都曾经使用过这样的架构，但随着业务数据量的不断增加，和访问量的持续增长，会遇到了很多问题：

　　1.MySQL需要不断进行拆库拆表，Memcached也需不断跟着扩容，扩容和维护工作占据大量开发时间。

　　2.Memcached与MySQL数据库数据一致性问题。

　　3.Memcached数据命中率低或down机，大量访问直接穿透到DB，MySQL无法支撑。

　　4.跨机房cache同步问题。

##2、NoSQL主要用于解决以下几种问题
1.少量数据存储，高速读写访问。此类产品通过数据全部in-momery 的方式来保证高速访问，同时提供数据落地的功能，实际这正是Redis最主要的适用场景。

2.海量数据存储，分布式系统支持，数据一致性保证，方便的集群节点添加/删除。

3.数据一致性保证。这方面最具代表性的是dynamo和bigtable 2篇论文所阐述的思路。前者是一个完全无中心的设计，节点之间通过gossip方式传递集群信息，数据保证最终一致性，后者是一个中心化的方案设计，通过类似一个分布式锁服务来保证强一致性,数据写入先写内存和redo log，然后定期compat归并到磁盘上，将随机写优化为顺序写，提高写入性能。

4.Schema free，auto-sharding等。比如目前常见的一些文档数据库都是支持schema-free的，直接存储json格式数据，并且支持auto-sharding等功能，比如mongodb。

##3、简单比较
如果简单地比较Redis与Memcached的区别，大多数都会得到以下观点：

1  Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等数据结构的存储。

2  Redis支持数据的备份，即master-slave模式的数据备份。

3  Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。

##4、 memcached和redis的比较
1 网络IO模型

　　Memcached是多线程，非阻塞IO复用的网络模型，分为监听主线程和worker子线程，监听线程监听网络连接，接受请求后，将连接描述字pipe 传递给worker线程，进行读写IO, 网络层使用libevent封装的事件库，多线程模型可以发挥多核作用，但是引入了cache coherency和锁的问题，比如，Memcached最常用的stats 命令，实际Memcached所有操作都要对这个全局变量加锁，进行计数等工作，带来了性能损耗。


　　Redis使用单线程的IO复用模型，自己封装了一个简单的AeEvent事件处理框架，主要实现了epoll、kqueue和select，对于单纯只有IO操作来说，单线程可以将速度优势发挥到最大，但是Redis也提供了一些简单的计算功能，比如排序、聚合等，对于这些操作，单线程模型实际会严重影响整体吞吐量，CPU计算过程中，整个IO调度都是被阻塞住的。

2.内存管理方面

　　Memcached使用预分配的内存池的方式，使用slab和大小不同的chunk来管理内存，Item根据大小选择合适的chunk存储，内存池的方式可以省去申请/释放内存的开销，并且能减小内存碎片产生，但这种方式也会带来一定程度上的空间浪费，并且在内存仍然有很大空间时，新的数据也可能会被剔除，原因可以参考Timyang的文章：http://timyang.net/data/Memcached-lru-evictions/

　　Redis使用现场申请内存的方式来存储数据，并且很少使用free-list等方式来优化内存分配，会在一定程度上存在内存碎片，Redis跟据存储命令参数，会把带过期时间的数据单独存放在一起，并把它们称为临时数据，非临时数据是永远不会被剔除的，即便物理内存不够，导致swap也不会剔除任何非临时数据(但会尝试剔除部分临时数据)，这点上Redis更适合作为存储而不是cache。

3.数据一致性问题

　　Memcached提供了cas命令，可以保证多个并发访问操作同一份数据的一致性问题。 Redis没有提供cas 命令，并不能保证这点，不过Redis提供了事务的功能，可以保证一串 命令的原子性，中间不会被任何操作打断。

4.存储方式及其它方面

　　Memcached基本只支持简单的key-value存储，不支持枚举，不支持持久化和复制等功能
　　Redis除key/value之外，还支持list,set,sorted set,hash等众多数据结构，提供了KEYS
　　进行枚举操作，但不能在线上使用，如果需要枚举线上数据，Redis提供了工具可以直接扫描其dump文件，枚举出所有数据，Redis还同时提供了持久化和复制等功能。

5.关于不同语言的客户端支持

　　在不同语言的客户端方面，Memcached和Redis都有丰富的第三方客户端可供选择，不过因为Memcached发展的时间更久一些，目前看在客户端支持方面，Memcached的很多客户端更加成熟稳定，而Redis由于其协议本身就比Memcached复杂，加上作者不断增加新的功能等，对应第三方客户端跟进速度可能会赶不上，有时可能需要自己在第三方客户端基础上做些修改才能更好的使用。
　　根据以上比较不难看出，当我们不希望数据被踢出，或者需要除key/value之外的更多数据类型时，或者需要落地功能时，使用Redis比使用Memcached更合适。

##5、总结
　　1.Redis使用最佳方式是全部数据in-memory。  
　　2.Redis更多场景是作为Memcached的替代者来使用。  
　　3.当需要除key/value之外的更多数据类型支持时，使用Redis更合适。  
　　4.当存储的数据不能被剔除时，使用Redis更合适。