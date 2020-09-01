---
layout: post
title: Kafka简介
category: Kafka
comments: false
---

# 一、Kafka简介

Kafka是一个分布式、高可靠、高吞吐的发布订阅消息系统。

它的最大的特性就是可以实时的处理大量数据以满足各种需求场景：比如基于hadoop的批处理系统、低延迟的实时系统、storm/Spark流式处理引擎，web/nginx日志、访问日志，消息服务等等，用scala语言编写，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。

## 1.1 Kafka的使用场景

- 日志收集：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。
- 消息系统：解耦生产者和消费者、缓存消息等。
- 用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。
- 运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
- 流式处理：比如spark streaming和storm
- 事件源

## 1.2 基本组成

Broker：Kafka节点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。

生产者：生产message发送到topic

消费者：订阅topic消费message, consumer作为一个线程来消费

消费群组（Consumer group）：在消费群组里的消费者共同消费一个主题的数据，每个分区最多只有一个消费者消费，如果消费者的数量大于分区的数量，那么多出来的消费者会被闲置（为了避免多个消费者线程去处理同一条数据，间接保证了高吞吐）。所以如果想同时对一个topic做消费的话，启动多个consumer group就可以了，但是要注意的是，这里的多个consumer的消费都必须是顺序读取partition里面的message。

Topic: Topic相当于传统消息系统MQ中的一个队列queue，producer端发送的message必须指定是发送到哪个topic，但是不需要指定topic下的哪个partition，因为kafka会把收到的message进行load balance，均匀的分布在这个topic下的不同的partition上（ hash(message) % [broker数量] ）。如果生成者发送的消息里带有Key，那么拥有同样key的消息会放在同一个partition上。

Partition: 用于存储topic的信息，同时用于支持并发读和高可用等特性（1）一个Topic的Partition数量大于等于Broker的数量，可以提高吞吐率。（2）同一个Partition的Replica尽量分散到不同的机器，高可用。主题的分区数量可以增加，但是不能减少，否则会导致消息乱序。可以删除整个主题，然后在新主题上减少分区数量。

Segment：partition物理上由多个segment组成，每个Segment存着message信息

Partition Replica：每个partition可以在其他的kafka broker节点上存副本，以便某个kafka broker节点宕机不会影响这个kafka集群。存replica副本的方式是按照kafka broker的顺序存。例如有5个kafka broker节点，某个topic有3个partition，每个partition存2个副本，那么partition1存broker1,broker2，partition2存broker2,broker3。。。以此类推。 replica副本数越高，系统虽然越稳定，但是回来带资源和性能上的下降；replica副本少的话，也会造成系统丢数据的风险。

Partition leader & follower: 分区也有选主的过程，主分区副本负责生成者消息写入。这些信息也是放置在ZK上。

Offset: 消费者用于标记在主题读取消息的偏移值。

# 二、Kafka的设计思想

## 2.1 Broker Leader的选举

Broker集群受到ZK管理，所有的Broker会向ZK注册一个临时节点，只有一个会注册成功，成为Controller，其他失败的将成为follower。如果当主Broker宕机，那么其他follower监听到临时节点下线的消息，就会去竞争成为新的Controller。

Kafka集群中broker之间的关系：不是主从关系，各个broker在集群中地位一样，我们可以随意的增加或删除任何一个broker节点。

## 2.2 什么是ISR
in-sync Replica: Leader中记录的与其保持同步的Replica的列表

当replica的消息落后太多、或者follower的延迟超过某个时长，就将replica从ISR中移出。

对于多数投票法则，是根据所有副本节点的状况动态的选择最适合的作为leader.Kafka并不是使用这种方法。ISR这个集合中的任何一个节点随时都可以被选为leader。

如果当所有的节点都down掉，kafka会选择所有节点中（不只是ISR）第一个恢复的节点作为leader.


简单来说，分区中的所有副本统称为 AR (Assigned Replicas), AR = ISR+OSR （Out-of-Sync Replied，与leader副本同步滞后过多的副本）

## 2.3 Consumer Rebalance
再均衡，就是调整partition上的consumer的分布。

触发条件：
（1）Consumer增加或删除会触发 Consumer Group的Rebalance（2）Broker的增加或者减少都会触发 Consumer Rebalance

## 2.4 offset提交时机
- auto：由kafka负责提交偏移量，这样不能保证消费者已经正确地处理过消息。
- 读取-commit-提交：也有同样的问题
- 处理完再commit：这种情况下Consumer端的响应会比较慢。

偏移量在旧版本（<0.9.x)保存在ZK；Kafka是会将偏移量存储在_consumer_offset的topic上。

## 2.5 消息是怎么存储在Kafka上的？
（1）producer先把message发送到partition leader，再由leader发送给其他partition follower。

（2）当收到消息的Replica的个数不少于配置的ACK，再向Producer发送ACK。当ACK的配置（request.required.acks）为0，表示立马反馈给生产者ACK，如果配置为1，表示只要主分区完成写入，就给生产者ACK，如果配置all(acks=-1)，表示等待其他副本都完成写入，但是对于性能会有所损耗。

（3）如果某个Replica不工作，如果它不在leader分区的ack列表，则毫无影响；如果在，那么会等消息的timeout，由leader来将它从ACK列表中除名。

## 2.6 Kafka是如何保证消息消费是可靠的

消息消费的可靠性，Kafka提供的是“At least once”模型，因为消息的读取进度由offset提供，offset可以由消费者自己维护也可以维护在zookeeper里，但是当消息消费后consumer挂掉，offset没有即时写回，就有可能发生重复读的情况，这种情况同样可以通过调整commit offset周期、阈值缓解，甚至消费者自己把消费和commit 
offset做成一个事务解决，但是如果你的应用不在乎重复消费，那就干脆不要解决，以换取最大的性能。

At most once（poll->commit offset->handle）: 得在消息上加唯一性标识，或依赖其他软件的唯一性保证（如mysql的unique）

poll->handle->commit offset 是 At least once.

恰好一次(exactly once)：最少1次＋消费者的输出中额外增加已处理消息最大编号：由于存在已处理消息最大编号，不会出现重复处理消息的情况。

## 2.7 为什么Kafka是高吞吐的？

Kafka的IO很快，首先，Kafka的数据的写入和读取均是顺序的，所以速度能达到o(1)。

另外，在返回数据的时候，采用了zero-copy的技术，直接从文件到网络，不经过中间缓存。（sendfile系统调用）

还有，Kafka还可以通过分批发送和压缩消息的机制，来保证更高速的数据传输。

kafka使用文件存储消息(append only log),这就直接决定kafka在性能上严重依赖文件系统的本身特性.

## 2.8 Kakfa的负载均衡
kafka集群中的任何一个broker,都可以向producer提供metadata信息,这些metadata中包含"集群中存活的servers列表"/"partitions leader列表"等信息(请参看zookeeper中的节点信息). 当producer获取到metadata信息之后, producer将会和Topic下所有partition  leader保持socket连接;消息由producer直接通过socket发送到broker,中间不会经过任何"路由层".


## 2.9 ZK在kafka中的作用

kafka使用zookeeper来存储一些meta信息,并使用了zookeeper watch机制来发现meta信息的变更并作出相应的动作(比如consumer失效,触发负载均衡等)

Broker node registry: 当一个kafka broker启动后,首先会向zookeeper注册自己的节点信息(临时znode),同时当broker和zookeeper断开连接时,此znode也会被删除.

Broker Topic Registry: 当一个broker启动时,会向zookeeper注册自己持有的topic和partitions信息,仍然是一个临时znode.

Consumer and Consumer group: 每个consumer客户端被创建时,会向zookeeper注册自己的信息;此作用主要是为了"负载均衡".一个group中的多个consumer可以交错的消费一个topic的所有partitions;简而言之,保证此topic的所有partitions都能被此group所消费,且消费时为了性能考虑,让partition相对均衡的分散到每个consumer上.（这是0.9之前的版本，之后的版本，消费群组保存在broker上）

Consumer id Registry: 每个consumer都有一个唯一的ID(host:uuid,可以通过配置文件指定,也可以由系统生成),此id用来标记消费者信息.

Consumer offset Tracking: 用来跟踪每个consumer目前所消费的partition中最大的offset.此znode为持久节点,可以看出offset跟group_id有关,以表明当group中一个消费者失效,其他consumer可以继续消费.

Partition Owner registry: 用来标记partition正在被哪个consumer消费.临时znode。此节点表达了"一个partition"只能被group下一个consumer消费,同时当group下某个consumer失效,那么将会触发负载均衡(即:让partitions在多个consumer间均衡消费,接管那些"游离"的partitions)

当consumer启动时,所触发的操作:

A) 首先进行"Consumer id Registry";

B) 然后在"Consumer id Registry"节点下注册一个watch用来监听当前group中其他consumer的"leave"和"join";只要此znode path下节点列表变更,都会触发此group下consumer的负载均衡.(比如一个consumer失效,那么其他consumer接管partitions).

C) 在"Broker id registry"节点下,注册一个watch用来监听broker的存活情况;如果broker列表变更,将会触发所有的groups下的consumer重新balance.

总结:

Producer端使用zookeeper用来"发现"broker列表,以及和Topic下每个partition leader建立socket连接并发送消息.

Broker端使用zookeeper用来注册broker信息,已经监测partition leader存活性.

Consumer端使用zookeeper用来注册consumer信息,其中包括consumer消费的partition列表等,同时也用来发现broker列表,并和partition leader建立socket连接,并获取消息。

# REF
> https://www.jianshu.com/p/734cf729d77b  

