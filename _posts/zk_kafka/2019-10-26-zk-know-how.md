---
layout: post
title: ZooKeeper
category: ZooKeeper
comments: false
---

# 一、什么是zookeeper?（is what）

## 1.1 介绍

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务（enables highly reliable distributed coordination），是Google的Chubby一个开源的实现，是Hadoop和Hbase的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。（可以简单地理解成保障数据管理中一致性）

原理：

ZooKeeper是以Fast Paxos算法为基础的，Paxos 算法存在活锁的问题，即当有多个proposer交错提交时，有可能互相排斥导致没有一个proposer能提交成功，而Fast Paxos作了一些优化，通过选举产生一个leader (领导者)，只有leader才能提交proposer，具体算法可见Fast Paxos。

ZooKeeper的基本运转流程：

1、选举Leader。  
2、同步数据。  
3、选举Leader过程中算法有很多，但要达到的选举标准是一致的。  
4、Leader要具有最高的执行ID，类似root权限。  
5、集群中大多数的机器得到响应并接受选出的Leader。  

## 1.2 特性
zookeeper有这样一个特性：【集群中只要有超过过半的机器是正常工作的，那么整个集群对外就是可用的】

也就是说如果有2个zookeeper，那么只要有1个死了zookeeper就不能用了，因为1没有过半，所以2个zookeeper的死亡容忍度为0；

同理，要是有3个zookeeper，一个死了，还剩下2个正常的，过半了，所以3个zookeeper的容忍度为1；

同理你多列举几个：2->0;3->1;4->1;5->2;6->2会发现一个规律，2n和2n-1的容忍度是一样的，都是n-1。所以官方推荐保持一个服务器数为奇数的集群。

## 1.3 相关算法和协议

paxos算法：

fast paxos算法：

Zab协议规定：来自Client的所有写请求，都要转发给ZK服务中唯一的Server—Leader，由Leader根据该请求发起一个Proposal（请求）。然后，其他的Server对该Proposal（请求）进行Vote（投票）。之后，Leader对Vote（投票）进行收集，当Vote数量过半时Leader会向所有的Server发送一个通知消息。最后，当Client所连接的Server收到该消息时，会把该操作更新到内存中并对Client的写请求做出回应。

为什么要有Observer？

为了在扩容时有更好的性能。因为如果全部的机器都参与到Zab协议中的投票中，由于Leader节点必须等待集群中过半Server响应投票，于是节点的增加使得部分计算机运行较慢，从而拖慢整个投票过程的可能性也随之提高，写操作也会随之下降。

# 二、构成 (has what)
ZooKeeper所提供的服务主要是通过：数据结构+原语+watcher机制，三个部分来实现的。

## 2.1 数据结构
ZooKeeper数据模型的结构整体上可以看作是一棵树，每个节点称做一个ZNode。每个ZNode都可以通过其路径唯一标识在每个ZNode上可存储少量数据。(默认是1M, 可以通过配置修改, 通常不建议在ZNode上存储大量的数据)

有四种类型的znode：

1. PERSISTENT-持久化目录节点

    客户端与zookeeper断开连接后，该节点依旧存在

2. PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点

    客户端与zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号

3. EPHEMERAL-临时目录节点

    客户端与zookeeper断开连接后，该节点被删除

4. EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点

    客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号

节点的类型在创建时即被确定，并且不能改变。

对于临时节点：该节点的生命周期依赖于创建它们的会话。一旦会话(Session)结束，临时节点将被自动删除，当然可以也可以手动删除。
虽然每个临时的Znode都会绑定到一个客户端会话，但他们对所有的客户端还是可见的。另外，ZooKeeper的临时节点不允许拥有子节点。


## 2.2 事务操作    
在ZooKeeper中，能改变ZooKeeper服务器状态的操作称为事务操作。一般包括数据节点创建与删除、数据内容更新和客户端会话创建与失效等操作。

对应每一个事务请求，ZooKeeper都会为其分配一个全局唯一的事务ID，用 ZXID 表示，通常是一个64位的数字。每一个 ZXID对应一次更新操作，从这些 ZXID 中可以间接地识别出 ZooKeeper 处理这些事务操作请求的全局顺序。

zxid是一个64位的数字。它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个 新的epoch，标识当前属于那个leader的统治时期。
低32位用于递增计数。

## 2.3 事件监听
ZooKeeper支持一种Watch操作，Client可以在某个ZNode上设置一个Watcher，来Watch该ZNode上的变化。如果该ZNode上有相应的变化，就会触发这个Watcher，把相应的事件通知给设置Watcher的Client。需要注意的是，ZooKeeper中的Watcher是一次性的，即触发一次就会被取消，如果想继续Watch的话，需要客户端重新设置Watcher。

## 2.4 节点状态/角色
每个集群中的节点都有一个状态 LOOKING, FOLLOWING, LEADING, OBSERVING。都属于这4种，每个节点启动的时候都是LOOKING状态，如果这个节点参与选举但最后不是leader，则状态是FOLLOWING，如果不参与选举则是OBSERVING，leader的状态是LEADING。

Zookeeper中的角色主要有以下三类，如下表所示:
![Zookeeper节点角色](/images/201910/zkrole.png)

系统模型如图所示：
![Zookeeper系统模型](/images/201910/zksystem.jpg)

## 2.5 Leader工作流程
Leader主要有三个功能：

1.恢复数据；

2.维持与Learner的心跳，接收Learner请求并判断Learner的请求消息类型；

3.Learner的消息类型主要有PING消息、REQUEST消息、ACK消息、REVALIDATE消息，根据不同的消息类型，进行不同的处理。

PING消息是指Learner的心跳信息；
REQUEST消息是Follower发送的提议信息，包括写请求及同步请求；
ACK消息是 Follower的对提议的回复，超过半数的Follower通过，则commit该提议； 
REVALIDATE消息是用来延长SESSION有效时间。

Leader的工作启动了三个线程来实现功能。

## 2.6 数据处理过程
Zookeeper节点数据操作流程:

![Zookeeper节点数据操作流程](/images/201910/zknode.png)

1.在Client向Follwer发出一个写的请求

2.Follwer把请求发送给Leader

3.Leader接收到以后开始发起投票并通知Follwer进行投票

4.Follwer把投票结果发送给Leader

5.Leader将结果汇总后如果需要写入，则开始写入同时把写入操作通知给Leader，然后commit;

6.Follwer把请求结果返回给Client

# 三、应用场景 (can do what)

## 3.1 分布式应用配置管理

假设我们的程序是分布式部署在多台机器上，如果我们要改变程序的配置文件，需要逐台机器去修改，非常麻烦，现在把这些配置全部放到zookeeper上去，保存在 zookeeper 的某个目录节点中，然后所有相关应用程序对这个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会收到 zookeeper 的通知，然后从 zookeeper 获取新的配置信息应用到系统中。

比如在HBase中，客户端就是连接一个Zookeeper，获得必要的HBase集群的配置信息，然后才可以进一步操作。

还有在开源的消息队列Kafka中，也使用Zookeeper来维护broker的信息。在Alibaba开源的SOA框架Dubbo中也广泛的使用Zookeeper管理一些配置来实现服务治理。

## 3.2 名字服务
类似于dns

## 3.3 分布式锁
见：[ZooKeeper实现分布式锁](/zk_kafka/2019/10/26/zk-ds-lock.html)

## 3.4 集群管理
常用。

# REF
> [zookeeper入门](https://blog.csdn.net/java_66666/article/details/81015302)  
> [Zookeeper 介绍 原理](https://www.cnblogs.com/centos2017/p/8118963.html)








