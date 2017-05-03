---
layout: post
title: Hadoop streaming
category: cloud
comments: false
---
## 1. Streaming简介 

Streaming框架允许任何程序语言实现的程序在Hadoop MapReduce中使用，方便已有程序向Hadoop平台移植。因此可以说对于hadoop的扩展性意义重大。

Streaming的原理是用Java实现一个包装用户程序的MapReduce程序，该程序负责调用MapReduce Java接口获取key/value对输入，创建一个新的进程启动包装的用户程序，将数据通过管道传递给包装的用户程序处理，然后调用MapReduce Java接口将用户程序的输出切分成key/value对输出。 

## 2. Streaming 优点 & 缺点
优点：

### 1.开发效率高，便于移植

只要按照标准输入输出格式进行编程，就可以满足hadoop要求。因此单机程序稍加改动就可以在集群上进行使用。 同样便于测试

只要按照 cat input | mapper | sort | reducer > output 进行单机测试即可。

如果单机测试通过，大多数情况是可以在集群上成功运行的，只要控制好内存就好了。

### 2 提高程序效率
有些程序对内存要求较高，如果用java控制内存毕竟不如C/C++。

## Streaming不足
1. Hadoop Streaming默认只能处理文本数据，无法直接对二进制数据进行处理   
2. Streaming中的mapper和reducer默认只能向标准输出写数据，不能方便地处理多路输出 