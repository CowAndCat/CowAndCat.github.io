---
layout: post
title: 分布式调研
category: arch
comments: false
---

## 一、Motivation
这段时间闲下来了，反思过去工作的两年，虽然经历了数个项目，但收获没达到自我预期。正好趁这段时间储备一些知识，以期望能用在后续的工作里。

偶然看到一个JD，其中一项：

    有大型分布式、高并发、高可用系统设计、开发和调优经验。例如akka集群、kafka、rocketMQ等消息中间件使用经验优先。

撇开语法错误，回忆起本科大三时选择的就是分布式方向，也摸过OpenStack，听过kafka、docker、spark。然而，现今的我却对这个方向不甚了解。

所以，趁现在充充电。

## 二、管中窥豹-kafka

Kafka是由Apache软件基金会开发的一个开源流处理平台，由Scala和Java编写。Kafka是一种高吞吐量的分布式发布订阅消息系统，它可以处理消费者规模的网站中的所有动作流数据。 

其具备如下特性：

- 通过O(1)的磁盘数据结构提供消息的持久化，这种结构对于即使数以TB的消息存储也能够保持长时间的稳定性能。
- 高吞吐量：即使是非常普通的硬件Kafka也可以支持每秒数百万的消息。
- 支持通过Kafka服务器和消费机集群来分区消息。
- 支持Hadoop并行数据加载

分四类核心API：Producer API，Consumer API, Streams API和Connector API
![kafka](/images/201804/kafka-apis.png "kafka apis")

相关术语：

- Broker:Kafka集群包含一个或多个服务器，这种服务器被称为broker.
- Topic: 每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。（物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处）
- Partition:Partition是物理上的概念，每个Topic包含一个或多个Partition.
- Producer:负责发布消息到Kafka broker
- Consumer:消息消费者，向Kafka broker读取消息的客户端。
- Consumer Group:每个Consumer属于一个特定的Consumer Group（可为每个Consumer指定group name，若不指定group name则属于默认的group）。

应用场景：

- Messaging: 做一个消息中间件，类似ActiveMQ or RabbitMQ,因为kafka具备低吞吐、低延迟和强持久性的特点。
- Website Activity Tracking: 网页活动追踪（追踪页面展示，搜索，点击等用户操作）
- Metrics: used for operational monitoring data. 当监控用。
- Log Aggregation: 日志整合。 相比较Scribe或Flume，Kafka性能更好（O(1)）,持久性更难高，延迟更低。
- Stream Processing: 流式处理（比较抽象），有点类似集中时间段进行批量处理。已衍生出Kafka Streams, 其他类似的产品有Apach Storm和Apache Samza.
- Event Sourcing: 事件溯源
- Commit Log: 类似日志数据库，用于恢复数据。类似 BookKeeper.




