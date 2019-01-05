---
layout: post
title: Java锁--Lock实现原理(底层实现)
category: tech
comments: false
---
## 一、synchronized的实现

在 HotSpot JVM实现中，锁有个专门的名字：对象监视器。 在java虚拟机中，每个对象和类在逻辑上都是和一个监视器相关联的。 

- 对于对象来说，相关联的监视器保护对象的实例变量。 
- 对于类来说，监视器保护类的类变量。

### 1.1 线程状态及状态转换

当多个线程同时请求某个对象监视器时，对象监视器会设置几种状态用来区分请求的线程：

- Contention List：所有请求锁的线程将被首先放置到该竞争队列
- Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
- Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
- OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
- Owner：获得锁的线程称为Owner
- !Owner：释放锁的线程

状态转换关系图：

!["monitor"](/images/201901/monitor-states.jpg)

新请求锁的线程将首先被加入到ConetentionList中，当某个拥有锁的线程（Owner状态）调用unlock之后，如果发现 EntryList为空则从ContentionList中移动线程到EntryList，下面说明下ContentionList和EntryList 的实现方式：

ContentionList并不是一个真正的Queue，而只是一个虚拟队列，原因在于ContentionList是由Node及其next指 针逻辑构成，并不存在一个Queue的数据结构。ContentionList是一个后进先出（LIFO）的队列，每次新加入Node时都会在队头进行， 通过CAS改变第一个节点的的指针为新增节点，同时设置新增节点的next指向后续节点，而取得操作则发生在队尾。显然，该结构其实是个Lock-Free的队列。

EntryList与ContentionList逻辑上同属等待队列，ContentionList会被线程并发访问，为了降低对 ContentionList队尾的争用，而建立EntryList。Owner线程在unlock时会从ContentionList中迁移线程到 EntryList，并会指定EntryList中的某个线程（一般为Head）为Ready（OnDeck）线程。Owner线程并不是把锁传递给 OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程需要重新竞争锁。这样做虽然牺牲了一定的公平性，但极大的提高了整体吞吐量，在 Hotspot中把OnDeck的选择行为称之为“竞争切换”。

### 1.2 总结

synchronized的底层实现主要依靠Lock-Free的队列，基本思路是自旋后阻塞，竞争切换后继续竞争锁，稍微牺牲了公平性，但获得了高吞吐量。

## 二、Lock的实现原理

经过观察ReentrantLock把所有Lock接口的操作都委派到一个Sync类上，该类继承了AbstractQueuedSynchronizer：

    static abstract class Sync extends AbstractQueuedSynchronizer  

Sync又有两个子类：

    final static class NonfairSync extends Sync       
    final static class FairSync extends Sync  

显然是为了支持公平锁和非公平锁而定义，默认情况下为非公平锁。

实现一个锁，主要需要考虑2个问题:

- 如何线程安全的修改锁状态位？
- 得不到锁的线程，如何排队？

### 2.1 修改锁状态位

现线程安全的关键在于：<font color="#dd0000">volatile变量</font>和<font color="#dd0000">CAS原语</font>的配合使用

### 2.2 排队

AbstractQueuedSynchronizer会把所有的请求线程构成一个CLH队列，当一个线程执行完毕（lock.unlock()）时会激活自己的后继节点，但正在执行的线程并不在队列中，而那些等待执行的线程全部处于阻塞状态，经过调查线程的显式阻塞是通过调用LockSupport.park()完成，而LockSupport.park()则调用 sun.misc.Unsafe.park()本地方法，再进一步，HotSpot在Linux中中通过调用pthread_mutex_lock函数把线程交给系统内核进行阻塞。

CLH队列：
!["CLH"](/images/201901/clh-lock.png)

获取不到锁的线程，会进入队尾，然后自旋，直到其前驱线程释放锁。

这样做的好处：假设有1000个线程等待获取锁，锁释放后，只会通知队列中的第一个线程去竞争锁，减少了并发冲突。（ZK的分布式锁，为了避免惊群效应，也使用了类似的方式：获取不到锁的线程只监听前一个节点）

原始CLH队列，一般用于实现自旋锁。而JUC(java.util.concurrent)中的实现，获取不到锁的线程，一般会时而阻塞，时而唤醒。

每个得不到锁的线程，都会讲自己封装成Node，加入队尾，或自旋或阻塞，直到获取锁。

### 三、对比

Lock的加锁和解锁都是由java代码配合native方法（调用操作系统的相关方法）实现的，而synchronize的加锁和解锁的过程是由JVM管理的

## REF
> [Lock的实现原理](https://www.cnblogs.com/duanxz/p/3559510.html)  
> [https://blog.csdn.net/tingfeng96/article/details/52219649](https://blog.csdn.net/tingfeng96/article/details/52219649)  
> [CLH队列锁](https://blog.csdn.net/aesop_wubo/article/details/7533186)  
> [ReentrantLock实现机制（CLH队列锁）](https://www.jianshu.com/p/b6efbdbdc6fa)