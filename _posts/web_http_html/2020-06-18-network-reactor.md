---
layout: post
title: Reactor模式
category: network
comments: false
--- 

# 一、Reactor模式

Reactor模式是一种典型的事件驱动的编程模型，Reactor逆置了程序处理的流程，其基本的思想即为Hollywood Principle— 'Don't call us, we'll call you'.

I/O多路复用可以用作并发事件驱动(event-driven)程序的基础，即整个事件驱动模型是一个状态机，包含了状态(state), 输入事件(input-event), 状态转移(transition), 状态转移即状态到输入事件的一组映射。通过I/O多路复用的技术检测事件的发生，并根据具体的事件(通常为读写)，进行不同的操作，即状态转移。

Reactor事件处理机制为：主程序将事件以及对应事件处理的方法在Reactor上进行注册, 如果相应的事件发生，Reactor将会主动调用事件注册的接口，即 回调函数. libevent即为封装了epoll并注册相应的事件(I/O读写，时间事件，信号事件)以及回调函数，实现的事件驱动的框架。

关键字：I/O多路复用，事件驱动，回调函数，非阻塞

## 1.1 组成
基于Reactor Pattern 处理模式中，定义以下三种角色:

Reactor 将I/O事件分派给对应的Handler

Acceptor 处理客户端新连接，并分派请求到处理器链中

Handlers 执行非阻塞读/写 任务


1.单Reactor单线程模型

最基本的单Reactor单线程模型（多个client+单个reactor+单个acceptor+多个handler）。其中Reactor线程，负责多路分离套接字，有新连接到来触发connect 事件之后，交由Acceptor进行处理，有IO读写事件之后交给hanlder 处理。

Acceptor主要任务就是构建handler ，在获取到和client相关的SocketChannel之后 ，绑定到相应的hanlder上，对应的SocketChannel有读写事件之后，基于racotor 分发,hanlder就可以处理了（所有的IO事件都绑定到selector上，有Reactor分发）。

该模型 适用于处理器链中业务处理组件能快速完成的场景。不过，这种单线程模型不能充分利用多核资源，所以实际使用的不多。

2.单Reactor多线程模型

相对于第一种单线程的模式来说，在处理业务逻辑，也就是获取到IO的读写事件之后，交由线程池来处理，这样可以减小主reactor的性能开销，从而更专注的做事件分发工作了，从而提升整个应用的吞吐。

（相当于reactor只做event分发的工作）

3.多Reactor多线程模型

第三种模型比起第二种模型，是将Reactor分成两部分，

1. mainReactor负责监听server socket，用来处理新连接的建立，将建立的socketChannel指定注册给subReactor。  
2. subReactor维护自己的selector, 基于mainReactor 注册的socketChannel多路分离IO读写事件，读写网 络数据，对业务处理的功能，另其扔给worker线程池来完成。

(面试的时候回答三要素：是什么，原理是什么，怎么使用)

# REF
> [Reactor模式](https://zhuanlan.zhihu.com/p/93612337)  
> [【NIO系列】——之Reactor模型](https://my.oschina.net/u/1859679/blog/1844109)
