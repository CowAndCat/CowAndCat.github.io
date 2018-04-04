---
layout: post
title: Hadoop 面试题
category: cloud
comments: false
---


海量数据正在不断生成，对于急需改变自己传统IT架构的企业而言，面对海量数据，如何分析并有效利用其价值，同时优化企业业务已成为现代企业转型过程中不可避免的问题。  

作为海量数据处理的一个重要工具——Hadoop也开始受到了越来越多人的关注。

Hadoop 是一种使用 Java 编写的分布式计算平台。它吸收了 Google 文件系统和 MapReduce 等产品的特性。

### 1、hadoop运行的原理?

hadoop主要由三方面组成:
1、HDFS
2、MapReduce
3、Hbase

![2](/images/201510/hadoop2.gif "hadoop2")  

- 在Hadoop的系统中，会有一台Master，主要负责NameNode的工作以及JobTracker的工作。  
- JobTracker的主要职责 就是启动、跟踪和调度各个Slave的任务执行。  
- 还会有多台Slave，每一台Slave通常具有DataNode的功能并负责TaskTracker的工作。  
- TaskTracker根据应用要求来结合本地数据执行Map任务以及Reduce任务。

### 2、map/Reduce的原理?

即“任务的分解与结果的汇总”。“Map（展开）”就是将一个任务分解成为多个任 务，“Reduce”就是将分解后多任务处理的结果汇总起来，得出最后的分析结果。


### 3、HDFS存储的机制?
- HDFS(Hadoop Distributed File System)默认的最基本的存储单位是64M的数据块。
- 和普通文件系统相同的是，HDFS中的文件是被分成64M一块的数据块存储的。
- 不同于普通文件系统的是，HDFS中，如果一个文件小于一个数据块的大小，并不占用整个数据块存储空间。

#### 3.1 NameNode、DataNode和 Client。  
![1](/images/201510/hadoop.gif "hadoop")  
上图中展现了整个HDFS三个重要角色：NameNode、DataNode和 Client。  

- NameNode可以看作是分布式文件系统中的管理者，**主要负责管理文件系统的命名空间、集群配置信息和存储块的复制**等。NameNode 会将文件系统的Meta-data存储在内存中，这些信息主要包括了文件信息、每一个文件对应的文件块的信息和每一个文件块在DataNode的信息等。
- **DataNode是文件存储的基本单元**，它将Block存储在本地文件系统中，保存了Block的Meta-data，同时周期性地将所有存在的 Block信息发送给NameNode。
- **Client就是需要获取分布式文件系统文件的应用程序**。

这里通过三个操作来说明他们之间的交互关系。

文件写入：

  a):Client向NameNode发起文件写入的请求。
  b):NameNode根据文件大小和文件块配置情况，返回给Client它所管理部分DataNode的信息。
  c):Client将文件划分为多个Block，根据DataNode的地址信息，按顺序写入到每一个DataNode块中。

文件读取：

  a):Client向NameNode发起文件读取的请求。
  b):NameNode返回文件存储的DataNode的信息。
  c):Client读取文件信息。

文件Block复制：

  a):NameNode发现部分文件的Block不符合最小复制数或者部分DataNode失效。
  b):通知DataNode相互复制Block。
  c):DataNode开始直接相互复制.

#### 3.1 元数据节点(Namenode)和数据节点(datanode)
- 元数据节点用来管理文件系统的命名空间
	- 其将所有的文件和文件夹的元数据保存在一个文件系统树中。
	- 这些信息也会在硬盘上保存成以下文件：命名空间镜像(namespace image)及修改日志(edit log)
	- 其还保存了一个文件包括哪些数据块，分布在哪些数据节点上。然而这些信息并不存储在硬盘上，而是在系统启动的时候从数据节点收集而成的。
	
- 数据节点是文件系统中真正存储数据的地方。
	- 客户端(client)或者元数据信息(namenode)可以向数据节点请求写入或者读出数据块。
	- 其周期性的向元数据节点回报其存储的数据块信息。

- 从元数据节点(secondary namenode)
	- 从元数据节点并不是元数据节点出现问题时候的备用节点，它和元数据节点负责不同的事情。
	- 其主要功能就是周期性将元数据节点的命名空间镜像文件和修改日志合并，以防日志文件过大。这点在下面会相信叙述。
	- 合并过后的命名空间镜像文件也在从元数据节点保存了一份，以防元数据节点失败的时候，可以恢复。

### 4、举一个简单的例子说明mapreduce是怎么来运行的 ?

### 5、面试的人给你出一些问题,让你用mapreduce来实现？
      比如:现在有10个文件夹,每个文件夹都有1000000个url.现在让你找出top1000000url。
### 6、hadoop中Combiner的作用?

一、作用  
1、**combiner最基本是实现本地key的聚合，对map输出的key进行排序，value进行迭代。**如下所示：
map: (K1, V1) → list(K2, V2) 
combine: (K2, list(V2)) → list(K2, V2) 
reduce: (K2, list(V2)) → list(K3, V3)

2、combiner还具有类似本地的reduce功能.
例如hadoop自带的wordcount的例子和找出value的最大值的程序，combiner和reduce完全一致。如下所示：
map: (K1, V1) → list(K2, V2) 
combine: (K2, list(V2)) → list(K3, V3) ，减轻reduce的负担！reduce: (K3, list(V3)) → list(K4, V4) 

3、如果不用combiner，那么，所有的结果都是reduce完成，效率会相对低下。使用combiner，先完成的map会在本地聚合，提升速度。

举一个hadoop自带的wordcount例子说明。
value就是一个叠加的数字，所以map一结束就可以进行reduce的value叠加，而不必要等到所有的map结束再去进行reduce的value叠加。

### 7、Map/Reduce中的Partiotioner使用
主要是想reduce的**结果能够根据key再次分类**输出到不同的文件夹中。

### 8、必须使用 Java 编写应用程序吗？
不。有几种办法让非 Java 代码与 Hadoop 协同工作。


- HadoopStreaming 允许用任何 shell 命令作为 map 或 reduce 函数。
- libhdfs 是一种基于 JNI 的 C 语言版 API（仅用于 HDFS）。
- Hadoop Pipes 是一种兼容 SWIG 的 C++ API （非 JNI），用于编写 MapReduce 作业。

### 9、[Hadoop优化](http://p-x1984.iteye.com/blog/1113410)
- 注重job重用, 主要是设计key和自定义OutputFormat, 将能合并的mapred job合并.
- 避免不必要的reduce任务.  
(1). 假定要处理的数据是排序且已经分区的. 或者对于一份数据, 需要多次处理, 可以先排序分区.  
(2). 自定义InputSplit, 将单个分区作为单个mapred的输入.  
(3). 在map中处理数据, Reducer设置为空.   
这样, 既重用了已有的 "排序", 也避免了多余的reduce任务.
- 大限度地重用对象, 避免对象的生成/销毁开销.  
该点在hadoop自带的org.apache.hadoop.mapred.MapRunner中尤为突出, 它使用同一个key对象和同一个value对象不停读取原始数据, 再将自身交给mapper处理.
- ……