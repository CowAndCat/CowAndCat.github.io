---
layout: post
title: nginx
category: arch
comments: false
---

## 1. nginx 架构

nginx采用多线程的方式。

在启动后，会有一个master进程和多个worker进程。master进程主要用来管理worker进程。包含：接收来自外界的信号，向各worker进程发送信号，监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程。而基本的网络事件，则是放在worker进程中来处理了。多个worker进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。

一个请求，只可能在一个worker进程中处理，一个worker进程，不可能处理其它进程的请求。worker进程的个数是可以设置的，一般我们会设置与机器cpu核数一致，这里面的原因与nginx的进程模型以及事件处理模型是分不开的。（更多的worker数，只会导致进程来竞争cpu资源了，从而带来不必要的上下文切换。而且，nginx为了更好的利用多核特性，提供了cpu亲缘性的绑定选项，我们可以将某一个进程绑定在某一个核上，这样就不会因为进程的切换带来cache的失效。）

nginx的进程模型：在请求到来的时候，每个work进程的listenfd在新链接到来的时候变得可读，为保证只有一个进程处理该连接，所有worker进程在注册listenfd读事件前抢accept_mutex，抢到互斥锁的那个进程注册listenfd读事件，在读事件里调用accept接受该连接。一个请求，完全由worker进程来处理，而且只在一个worker进程中处理。

事件处理模型：采用异步非阻塞的方式来处理。当一堆请求过来的时候，每一个worker只处理一个请求，未占到资源的请求不断的尝试创建事件，

题外话：apache处理高并发：每个请求会独占一个工作线程，当并发数上到几千时，就同时有几千的线程在处理请求了。这对操作系统来说，是个不小的挑战，线程带来的内存占用非常大，线程的上下文切换带来的cpu开销很大，自然性能就上不去了，而这些开销完全是没有意义的。

对于一个基本的web服务器来说，事件通常有三种类型，网络事件、信号、定时器。从上面的讲解中知道，网络事件通过异步非阻塞可以很好的解决掉。如何处理信号与定时器？ 详见 [Click](http://tengine.taobao.org/book/chapter_02.html#id1)

## 2. nginx 基础概念

### 2.1 connection

nginx是如何处理一个连接的?

首先，nginx在启动时，会解析配置文件，得到需要监听的端口与ip地址，然后在nginx的master进程里面，先初始化好这个监控的socket(创建socket，设置addrreuse等选项，绑定到指定的ip地址端口，再listen)，然后再fork出多个子进程出来，然后子进程会竞争accept新的连接。此时，客户端就可以向nginx发起连接了。当客户端与服务端通过三次握手建立好一个连接后，nginx的某一个子进程会accept成功，得到这个建立好的连接的socket，然后创建nginx对连接的封装，即ngx_connection_t结构体。接着，设置读写事件处理函数并添加读写事件来与客户端进行数据的交换。最后，nginx或客户端来主动关掉连接，到此，一个连接就寿终正寝了。

worker_connections 参数：  
很多人会误解worker_connections这个参数的意思，认为这个值就是nginx所能建立连接的最大值。其实不然，这个值是表示每个worker进程所能建立连接的最大值，所以，一个nginx能建立的最大连接数，应该是worker_connections * worker_processes。而如果是HTTP作为反向代理来说，最大并发数量应该是worker_connections * worker_processes/2。因为作为反向代理服务器，每个并发会建立与客户端的连接和与后端服务的连接，会占用两个连接。

### 2.2 request
ngx_http_request_t是对一个http请求的封装。一个http请求，包含请求行、请求头、请求体、响应行、响应头、响应体。

nginx会将请求头放在一个buffer里。(这个动作会在pipeline的时候省去一个RTT的时间)

keepalive、pipe、lingering_close 从略。

## 3. nginx的请求处理

nginx使用一个多进程模型来对外提供服务，其中一个master进程，多个worker进程。master进程负责管理nginx本身和其他worker进程。

所有实际上的业务处理逻辑都在worker进程。worker进程中有一个函数，执行无限循环，不断处理收到的来自客户端的请求，并进行处理，直到整个nginx服务被停止。

worker进程中，ngx_worker_process_cycle()函数就是这个无限循环的处理函数。在这个函数中，一个请求的简单处理流程如下：

- 1. 操作系统提供的机制（例如epoll, kqueue等）产生相关的事件。
- 2. 接收和处理这些事件，如是接受到数据，则产生更高层的request对象。
- 3. 处理request的header和body。
- 4. 产生响应，并发送回客户端。
- 5. 完成request的处理。
- 6. 重新初始化定时器及其他事件。

## 4. 常用命令

**停止操作**

步骤1：查询nginx主进程号

    ps -ef | grep nginx
在进程列表里 面找master进程，它的编号就是主进程号了。

步骤2：发送信号
从容停止Nginx：

    kill -QUIT 主进程号
快速停止Nginx：

    kill -TERM 主进程号
强制停止Nginx：

    pkill -9 nginx

**平滑重启**
平滑重启命令：

    kill -HUP 住进程号或进程号文件路径
或者使用

    nginx -s reload

**判断Nginx配置是否正确**

    nginx -t -c /usr/nginx/conf/nginx.conf
或者

    nginx -t （推荐）

## 参考
> [http://tengine.taobao.org/book/](http://tengine.taobao.org/book/)