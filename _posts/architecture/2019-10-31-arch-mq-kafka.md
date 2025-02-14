---
layout: post
title: 消息中间件kafka
category: arch
comments: false
---
# 一、kafka
pache Kafka® 是一个分布式流处理平台. 它可以水平扩展，高可用，速度快，并且已经运行在数千家公司的生产环境。

可用于构建实时的数据管道和流式两大类别的应用:

- 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于message queue)
- 构建实时流式应用程序，对这些流数据进行转换或者影响。 (就是流处理，通过kafka stream topic和topic之间内部进行变化)

## 1.1 四个核心API
- The Producer API 允许一个应用程序发布一串流式的数据到一个或者多个Kafka topic。
- The Consumer API 允许一个应用程序订阅一个或多个 topic ，并且对发布给他们的流式数据进行处理。
- The Streams API 允许一个应用程序作为一个流处理器，消费一个或者多个topic产生的输入流，然后生产一个输出流到一个或多个topic中去，在输入输出流中进行有效的转换。
- The Connector API 允许构建并运行可重用的生产者或者消费者，将Kafka topics连接到已存在的应用程序或者数据系统。比如，连接到一个关系型数据库，捕捉表（table）的所有变更内容。

在Kafka中，客户端和服务器使用一个简单、高性能、支持多语言的 TCP 协议.

## 1.2 topics
Kafka 通过 topic 对存储的流数据进行分类。一个topic是用于发布记录的一种类别或推送名字，在Kafka中，一个topic通常有多个订阅者。

对于每一个topic， Kafka集群都会维持一个分区日志，如下所示：

![anatomy](/images/201910/log_anatomy.png "kafka topic")

每个分区都是有序且顺序不可变的记录集，并且不断地追加到结构化的commit log文件。分区中的每一个记录都会分配一个id号来表示顺序，我们称之为offset，offset用来唯一的标识分区中每一条记录。

Kafka 集群保留所有发布的记录—无论他们是否已被消费—并通过一个可配置的参数——保留期限来控制. 举个例子， 如果保留策略设置为2天，一条记录发布后两天内，可以随时被消费，两天过后这条记录会被抛弃并释放磁盘空间。Kafka的性能和数据大小无关，所以长时间存储数据没有什么问题.

![log_consumer](/images/201910/log_consumer.png "kafka consumer")
事实上，在每一个消费者中唯一保存的元数据是offset（偏移量）,即消费者在log中的位置.

偏移量由消费者所控制: 通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。

这些细节说明Kafka 消费者是非常廉价的—消费者的增加和减少，对集群或者其他消费者没有多大的影响。比如，你可以使用命令行工具，对一些topic内容执行 tail操作，并不会影响已存在的消费者消费数据。

日志中的 partition（分区）有以下几个用途。第一，当日志大小超过了单台服务器的限制，允许日志进行扩展。第二，它们会作为并行系统的一个单元（组成一个并行系统，比如具备leader-follower分布机制、地理备份等特性）

## 1.3 分布式
日志的分区partition （分布）在Kafka集群的服务器上。每个服务器在处理数据和请求时，共享这些分区。每一个分区都会在已配置的服务器上进行备份，确保容错性.

每个分区都有一台 server 作为 “leader”，零台或者多台server作为 follwers 。leader server 处理一切对 partition （分区）的读写请求，而follwers只需被动的同步leader上的数据。当leader宕机了，followers 中的一台服务器会自动成为新的 leader。每台 server 都会成为某些分区的 leader 和某些分区的 follower，因此集群的负载是平衡的。

## 1.4 生产者
生产者可以将数据发布到所选择的topic（主题）中。生产者负责将记录分配到topic的哪一个 partition（分区）中。可以使用循环的方式来简单地实现负载均衡，也可以根据某些语义分区函数(例如：记录中的key)来完成。下面会介绍更多关于分区的使用。

## 1.5 消费者
消费者使用一个 消费组 名称来进行标识，发布到topic中的每条记录被分配给订阅消费组中的一个消费者实例.消费者实例可以分布在多个进程中或者多个机器上。

如果所有的消费者实例在同一消费组中，消息记录会负载平衡到每一个消费者实例.

如果所有的消费者实例在不同的消费组中，每条消息记录会广播到所有的消费者进程.

# 二、kafka的优势
## 2.1 作为消息系统

传统的消息系统有两个模块: 队列 和 发布-订阅。 在队列中，消费者池从server读取数据，每条记录被池子中的一个消费者消费; 在发布订阅中，记录被广播到所有的消费者。

两者均有优缺点。 队列的优点在于它允许你将处理数据的过程分给多个消费者实例，使你可以扩展处理过程。 不好的是，队列不是多订阅者模式的—一旦一个进程读取了数据，数据就会被丢弃。 

而发布-订阅系统允许你广播数据到多个进程，但是无法进行扩展处理，因为每条消息都会发送给所有的订阅者。

消费组能让Kafka兼顾两者的优势。在队列中，消费组允许你将处理过程分发给一系列进程(消费组中的成员)。 在发布订阅中，Kafka允许你将消息广播给多个消费组。

Kafka的优势在于每个topic都有以下特性—可以扩展处理并且允许多订阅者模式—不需要只选择其中一个.

另外，Kafka相比于传统消息队列还具有更严格的顺序保证。

传统队列在服务器上保存有序的记录，如果多个消费者消费队列中的数据， 服务器将按照存储顺序输出记录。 虽然服务器按顺序输出记录，但是记录被异步传递给消费者， 因此记录可能会无序的到达不同的消费者。这意味着在并行消耗的情况下， 记录的顺序是丢失的。因此消息系统通常使用“唯一消费者”的概念，即只让一个进程从队列中消费， 但这就意味着不能够并行地处理数据。

Kafka 设计的更好。topic中的partition是一个并行的概念。 Kafka能够为一个消费者池提供顺序保证和负载平衡，是通过将topic中的partition分配给消费者组中的消费者来实现的， 以便每个分区由消费组中的一个消费者消耗。通过这样，我们能够确保消费者是该分区的唯一读者，并按顺序消费数据。 众多分区保证了多个消费者实例间的负载均衡。但请注意，消费者组中的消费者实例个数不能超过分区的数量。

## 2.2 Kafka 作为存储系统
许多消息队列可以发布消息，除了消费消息之外还可以充当中间数据的存储系统。那么Kafka作为一个优秀的存储系统有什么不同呢?

数据写入Kafka后被写到磁盘，并且进行备份以便容错。直到完全备份，Kafka才让生产者认为完成写入，即使写入失败Kafka也会确保继续写入

Kafka使用磁盘结构，具有很好的扩展性—50kb和50TB的数据在server上表现一致。

可以存储大量数据，并且可通过客户端控制它读取数据的位置，您可认为Kafka是一种高性能、低延迟、具备日志存储、备份和传播功能的分布式文件系统。

## 2.3 Kafka用做流处理
Kafka 流处理不仅仅用来读写和存储流式数据，它最终的目的是为了能够进行实时的流处理。

在Kafka中，流处理器不断地从输入的topic获取流数据，处理数据后，再不断生产流数据到输出的topic中去。

例如，零售应用程序可能会接收销售和出货的输入流，经过价格调整计算后，再输出一串流式数据。

简单的数据处理可以直接用生产者和消费者的API。对于复杂的数据变换，Kafka提供了Streams API。 Stream API 允许应用做一些复杂的处理，比如将流数据聚合或者join。

这一功能有助于解决以下这种应用程序所面临的问题：处理无序数据，当消费端代码变更后重新处理输入，执行有状态计算等。

Streams API建立在Kafka的核心之上：它使用Producer和Consumer API作为输入，使用Kafka进行有状态的存储， 并在流处理器实例之间使用相同的消费组机制来实现容错。

# REF 
> [Apache kafka中文手册](http://www.kubiji.cn/book/apache_kafka/)  
> [kafka中文文档](http://kafka.apachecn.org/)