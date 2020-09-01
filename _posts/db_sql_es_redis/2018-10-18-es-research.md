---
layout: post
title: ElasticSearch 探索
category: ES
comments: false
---

# 一、 ElasticSearch简介

可扩展的全文搜索+实时文档存储。

ElasticSearch是一个基于Lucene的搜索服务器(但是，你没法直接用 Lucene，必须自己写代码去调用它的接口。Elastic 内部使用 Lucene 做索引与搜索，但是它的目的是使全文检索变得简单， 通过隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API，开箱即用。)。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

根据DB-Engines的排名显示，Elasticsearch是最受欢迎的企业搜索引擎，其次是Apache Solr，也是基于Lucene。

它可以快速地储存、搜索和分析海量数据。维基百科、Stack Overflow、Github 都采用它。在统计以及日志类时间序的数据的存储和分析以及可视化方面，是引领者。

国内：百度（在casio、云分析、网盟、预测、文库、直达号、钱包、风控等业务上都应用了ES，单集群每天导入30TB+数据，总共每天60TB+）、新浪（见大数据架构--log），阿里巴巴、腾讯等公司均有对ES的使用。

使用比较广泛的平台ELK(ElasticSearch, Logstash, Kibana)

与传统的关系型数据库还是有较大的区别。

- 一个分布式的实施文档存储，每个字段可以被索引与搜索
- 一个分布式实时分析搜索引擎
- 可以胜任上百个服务节点的扩展，并支持PB级别的结构化或者非结构化数据。

一线公司ES使用场景：

1）新浪ES 如何分析处理32亿条实时日志 [http://dockone.io/article/505](http://dockone.io/article/505)  
2）阿里ES 构建挖财自己的日志采集和分析体系 [http://afoo.me/columns/tec/logging-platform-spec.html](http://afoo.me/columns/tec/logging-platform-spec.html)   
3）有赞ES 业务日志处理 [http://tech.youzan.com/you-zan-tong-ri-zhi-ping-tai-chu-tan/](http://tech.youzan.com/you-zan-tong-ri-zhi-ping-tai-chu-tan/)   
4）ES实现站内搜索 [http://www.wtoutiao.com/p/13bkqiZ.html](http://www.wtoutiao.com/p/13bkqiZ.html)  

es 中文社区：[https://elasticsearch.cn/](https://elasticsearch.cn/)

# 二、相关概念 （模块结构）

![1](/images/201809/es-structure.png "模块结构")

### 2.1 cluster

代表一个集群，集群中有多个节点(Node)，其中有一个为主节点，这个主节点是可以通过选举产生的，主从节点是对于集群内部来说的。es的一个概念就是去中心化，字面上理解就是无中心节点，这是对于集群外部来说的，因为从外部来看es集群，在逻辑上是个整体，你与任何一个节点的通信和与整个es集群通信是等价的。

### 2.2 shards

代表索引分片，es可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改。

### 2.3 replicas

代表索引副本，es可以设置多个索引的副本，副本的作用一是提高系统的容错性，当某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高es的查询效率，es会自动对搜索请求进行负载均衡。

### 2.4 recovery

代表数据恢复或叫数据重新分布，es在有节点加入或退出时会根据机器的负载对索引分片进行重新分配，挂掉的节点重新启动时也会进行数据恢复。

### 2.5 river

代表es的一个数据源，也是其它存储方式（如：数据库）同步数据到es的一个方法。它是以插件方式存在的一个es服务，通过读取river中的数据并把它索引到es中，官方的river有couchDB的，RabbitMQ的，Twitter的，Wikipedia的。

### 2.6 gateway

代表es索引快照的存储方式，es默认是先把索引存放到内存中，当内存满了时再持久化到本地硬盘。gateway对索引快照进行存储，当这个es集群关闭再重新启动时就会从gateway中读取索引备份数据。

es支持多种类型的gateway，有本地文件系统（默认），分布式文件系统，Hadoop的HDFS和amazon的s3云存储服务。

### 2.7 discovery.zen
代表es的自动发现节点机制，es是一个基于p2p的系统，它先通过广播寻找存在的节点，再通过多播协议来进行节点之间的通信，同时也支持点对点的交互。

### 2.8 Transport

代表es内部节点或集群与客户端的交互方式，默认内部是使用tcp协议进行交互，同时它支持http协议（json格式）、thrift、servlet、memcached、zeroMQ等的传输协议（通过插件方式集成）。

# 三、相关概念 （内容组织）

Mysql与Elasticsearch核心概念对比示意图:

![1](/images/201809/es-term-mapping.jpg "es-term-mapping")

### 3.1 索引（Index)

索引与关系型数据库实例(Database)相当。索引只是一个 逻辑命名空间，它指向一个或多个分片(shards)，内部用Apache Lucene实现索引中数据的读写。

### 3.2 文档类型（Type）

相当于数据库中的table概念。每个文档在ElasticSearch中都必须设定它的类型。文档类型使得同一个索引中在存储结构不同文档时，只需要依据文档类型就可以找到对应的参数映射(Mapping)信息，方便文档的存取。

### 3.3 字段（field）

在索引完了之后，是类型，类型之后，是字段.

保存的某个表里面的属性的的名称，比如，你存个学生类型，会有个Student类型，这个学生对象是不是可以有name，age，class等属性。这些属性在es里面就叫做。字段。

### 3.4 文档（Document)

相当于数据库中的row， 是可以被索引的基本单位。例如，你可以有一个的客户文档，有一个产品文档，还有一个订单的文档。文档是以JSON格式存储的。在一个索引中，您可以存储多个的文档。请注意，虽然在一个索引中有多分文档，但这些文档的结构是一致的，并在第一次存储的时候指定, 文档属于一种 类型(type)，各种各样的类型存在于一个 索引 中。

你也可以通过类比传统的关系数据库得到一些大致的相似之处：

    关系数据库       ⇒ 数据库  ⇒ 表    ⇒ 行    ⇒ 列(Columns)
    Elasticsearch   ⇒ 索引   ⇒ 类型  ⇒ 文档   ⇒ 字段(Fields)


一个文档不仅包含它的数据，也包含了元数据，元数据包括了有关文档的信息，有三个必须的元数据元素。\_index（文档在哪存放） \_type（文档表示的对象类别） \_id（文档唯一标识）

### 3.5 映射（Mapping）

相当于数据库中的schema，用来约束字段的类型，不过 Elasticsearch 的 mapping 可以自动根据数据创建。

# 四、Kibana 可视化分析

Kibana 是一个开源的分析和可视化平台，旨在与 Elasticsearch 合作。Kibana 提供搜索、查看和与存储在 Elasticsearch 索引中的数据进行交互的功能。开发者或运维人员可以轻松地执行高级数据分析，并在各种图表、表格和地图中可视化数据。

# 五、Logstash 基础入门

Logstash 是一个开源的数据收集引擎，它具有备实时数据传输能力。它可以统一过滤来自不同源的数据，并按照开发者的制定的规范输出到目的地。

logback能通过配置将日志append到kafka。

elasticsearch+Logstash+kibana 合成ELK。(整一套软件可以当作一个MVC模型，logstash是controller层，Elasticsearch是一个model层，kibana是view层。)

# 六、 ES必要的插件

必要的Head、kibana、IK（中文分词）、graph等插件的详细安装和使用。 [https://blog.csdn.net/column/details/deep-elasticsearch.html](https://blog.csdn.net/column/details/deep-elasticsearch.html)

## REF

> [Elasticsearch增、删、改、查操作深入详解](https://blog.csdn.net/laoyang360/article/details/51931981?utm_source=blogxgwz2)  
> [Logstash 基础入门](https://www.extlight.com/2017/10/30/Logstash-%E5%9F%BA%E7%A1%80%E5%85%A5%E9%97%A8/)  
> [elasticsearch性能测试工具rally深入详解](https://blog.csdn.net/laoyang360/article/details/52155481)  
> [Elasticsearch 权威指南（中文版）](https://es.xiaoleilu.com/)


