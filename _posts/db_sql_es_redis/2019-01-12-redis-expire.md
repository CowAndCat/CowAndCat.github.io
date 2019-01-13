---
layout: post
title: Redis基础知识
category: redis
comments: false
---

## 一、过期Keys处理
### 1.1 Redis如何淘汰过期的keys

Redis keys过期有两种方式：被动和主动方式。

当一些客户端尝试访问它时，key会被发现并主动的过期。

当然，这样是不够的，因为有些过期的keys，永远不会访问他们。 无论如何，这些keys应该过期，所以定时随机测试设置keys的过期时间。所有这些过期的keys将会从密钥空间删除。

具体就是Redis每秒10次做的事情：

- 1.测试随机的20个keys进行相关过期检测。
- 2.删除所有已经过期的keys。
- 3.如果有多于25%的keys过期，重复步骤1.

这是一个平凡的概率算法，基本上的假设是，我们的样本是这个密钥控件，并且我们不断重复过期检测，直到过期的keys的百分百低于25%，这意味着，在任何给定的时刻，最多会清除1/4的过期keys。

### 1.2 在复制AOF文件时如何处理过期

为了获得正确的行为而不牺牲一致性，当一个key过期，DEL将会随着AOF文字一起合成到所有附加的slaves。在master实例中，这种方法是集中的，并且不存在一致性错误的机会。

然而，当slaves连接到master时，不会独立过期keys（会等到master执行DEL命令），他们仍然会在数据集里面存在，所以当slave当选为master时淘汰keys会独立执行，然后成为master。

## REF
> [Redis过期：EXPIRE key seconds](http://redis.cn/commands/expire.html)