---
layout: post
title: Redisson
category: redis
comments: false
---

## 一、介绍
Redisson是架设在Redis基础上的一个Java驻内存数据网格（In-Memory Data Grid）。【Redis官方推荐】

Redisson在基于NIO的Netty框架上，充分的利用了Redis键值数据库提供的一系列优势，在Java实用工具包中常用接口的基础上，为使用者提供了一系列具有分布式特性的常用工具类。使得原本作为协调单机多线程并发程序的工具包获得了协调分布式多机多线程并发系统的能力，大大降低了设计和研发大规模分布式系统的难度。同时结合各富特色的分布式服务，更进一步简化了分布式环境中程序相互之间的协作。

### 1.1 支持的重要功能
- 1.支持云托管服务模式（同时支持亚马逊云的ElastiCache Redis和微软云的Azure Redis Cache）:
    - 自动发现主节点变化
- 2.支持Redis集群模式（同时支持亚马逊云的ElastiCache Redis Cluster和微软云的Azure Redis Cache）:
    - 自动发现主从节点
    - 自动更新状态和组态拓扑
    - 自动发现槽的变化
- 3.支持Redis哨兵模式:
    - 自动发现主、从和哨兵节点
    - 自动更新状态和组态拓扑
- 4.支持Redis主从模式
- 5.支持Redis单节模式
- 6.多节点模式均支持读写分离：从读主写，主读主写，主从混读主写
- 7.所有对象和接口均支持异步操作
- 8.自行管理的弹性异步连接池
- 9.所有操作线程安全
- 10.提供分布式对象  
通用对象桶（Object Bucket）、二进制流（Binary Stream）、地理空间对象桶（Geospatial Bucket）、BitSet、原子整长形（AtomicLong）、原子双精度浮点数（AtomicDouble）、话题（订阅分发）、 布隆过滤器（Bloom Filter）和基数估计算法（HyperLogLog）

- 11.提供分布式集合  
映射（Map）、多值映射（Multimap）、集（Set）、列表（List）、有序集（SortedSet）、计分排序集（ScoredSortedSet）、字典排序集（LexSortedSet）、列队（Queue）、双端队列（Deque）、阻塞队列（Blocking Queue）、有界阻塞列队（Bounded Blocking Queue）、 阻塞双端列队（Blocking Deque）、阻塞公平列队（Blocking Fair Queue）、延迟列队（Delayed Queue）、优先队列（Priority Queue）和优先双端队列（Priority Deque）

- 12.提供分布式锁和同步器  
可重入锁（Reentrant Lock）、公平锁（Fair Lock）、联锁（MultiLock）、 红锁（RedLock）、读写锁（ReadWriteLock）、信号量（Semaphore）、可过期性信号量（PermitExpirableSemaphore）和闭锁（CountDownLatch）

- 13.提供分布式服务  
分布式远程服务（Remote Service, RPC）、分布式实时对象（Live Object）服务、分布式执行服务（Executor Service）、分布式调度任务服务（Scheduler Service）和分布式映射归纳服务（MapReduce）
- 14.支持Spring框架
- 15.支持采用多种方式自动序列化和反序列化（Jackson JSON, Avro, Smile, CBOR, MsgPack, Kryo, FST, LZ4, Snappy和JDK序列化）

## 二、redisson配置解析

可以参考：[https://github.com/redisson/redisson/wiki/2.-%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95#21-%E7%A8%8B%E5%BA%8F%E5%8C%96%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95](https://github.com/redisson/redisson/wiki/2.-%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95#21-%E7%A8%8B%E5%BA%8F%E5%8C%96%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95)

一个redisson.yml例子：

    singleServerConfig:
      idleConnectionTimeout: 10000
      pingTimeout: 1000
      connectTimeout: 1000      # 默认值：10000ms
      timeout: 200              # 默认值：3000ms
      retryAttempts: 3
      retryInterval: 500        # 默认值：1500ms
      reconnectionTimeout: 500  # 默认值：3000ms
      failedAttempts: 3
      password: null
      subscriptionsPerConnection: 5
      clientName: null
      address: "redis://127.0.0.1:6379"
      subscriptionConnectionMinimumIdleSize: 0  # 默认值：1
      subscriptionConnectionPoolSize: 0
      connectionMinimumIdleSize: 20             # 默认值：32
      connectionPoolSize: 250                   # 默认值：64
      database: 0
      dnsMonitoring: false
      dnsMonitoringInterval: 5000
    threads: 0
    nettyThreads: 0
    codec: !<org.redisson.codec.JsonJacksonCodec> {}
    useLinuxNativeEpoll: false

- idleConnectionTimeout（连接空闲超时，单位：毫秒）默认值：10000
- slots（分片数量）默认值： 231，用于指定数据分片过程中的分片数量。支持数据分片/框架结构有：集（Set）、映射（Map）、BitSet、Bloom filter, Spring Cache和Hibernate Cache等.
- connectTimeout（连接超时，单位：毫秒）默认值：10000，同任何节点建立连接时的等待超时。时间单位是毫秒。
- timeout（命令等待超时，单位：毫秒）默认值：3000，等待节点回复命令的时间。该时间从命令发送成功时开始计时。
- retryInterval（命令重试发送时间间隔，单位：毫秒）默认值：1500，在一条命令发送失败以后，等待重试发送的时间间隔。时间单位是毫秒。
- reconnectionTimeout（重新连接时间间隔，单位：毫秒）默认值：3000，当与某个节点的连接断开时，等待与其重新建立连接的时间间隔。时间单位是毫秒。
- failedAttempts（执行失败最大次数）默认值：3，在某个节点执行相同或不同命令时，连续 失败 failedAttempts（执行失败最大次数） 时，该节点将被从可用节点列表里清除，直到 reconnectionTimeout（重新连接时间间隔） 超时以后再次尝试。
- password（密码）默认值：null，用于节点身份验证的密码。
- subscriptionsPerConnection（单个连接最大订阅数量）默认值：5，每个连接的最大订阅数量。
- clientName（客户端名称）默认值：null，在Redis节点里显示的客户端名称。
- address（节点地址）可以通过host:port的格式来指定节点地址。
- subscriptionConnectionMinimumIdleSize（发布和订阅连接的最小空闲连接数）默认值：1，用于发布和订阅连接的最小保持连接数（长连接）。Redisson内部经常通过发布和订阅来实现许多功能。长期保持一定数量的发布订阅连接是必须的。
- subscriptionConnectionPoolSize（发布和订阅连接池大小）默认值：50，用于发布和订阅连接的连接池最大容量。连接池的连接数量自动弹性伸缩。
- connectionMinimumIdleSize（最小空闲连接数）默认值：32，最小保持连接数（长连接）。长期保持一定数量的连接有利于提高瞬时写入反应速度。
- connectionPoolSize（连接池大小）默认值：64，连接池最大容量。连接池的连接数量自动弹性伸缩。
- database（数据库编号）默认值：0，尝试连接的数据库编号。
- dnsMonitoring（是否启用DNS监测）默认值：false，在启用该功能以后，Redisson将会监测DNS的变化情况。
- dnsMonitoringInterval（DNS监测时间间隔，单位：毫秒）默认值：5000，监测DNS的变化情况的时间间隔。

## REF
>[https://github.com/redisson/redisson/wiki](https://github.com/redisson/redisson/wiki)