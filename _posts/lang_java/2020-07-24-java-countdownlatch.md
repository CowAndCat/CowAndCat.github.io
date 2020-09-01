---
layout: post
title: Java CountDownLatch & CyclicBarrier同步屏障
category: java
comments: false
---

# 基本概念
## CyclicBarrier同步屏障

CyclicBarrier默认的构造方法CyclicBarrier(int parties)，其参数表示屏障拦截的线程数量，  
每个线程调用await方法告诉CyclicBarrier我已经到达屏障，然后当前线程被阻塞， 
直到被拦截的线程全部都到达了屏障，然后前面被阻塞的线程才能开始执行，否则会被一直阻塞。  
就像正式场合围着桌子吃饭，要等所有人都到到齐之后，才可以开餐。先来的人要等待。

查看栅栏类可以看到其主要是由ReentrantLock和Condition结合计数器实现。

##  Semaphore
信号量用于控制线程的并发数量，线程只有拿到许可证的时候才能执行。其源码中通过Sync同步器实现了AQS的共享模式。    
比喻：吃饭的时候，有一碗汤，只有两个勺子，也就是说同时只能由两个人拿到勺子成汤，其他人没拿到勺子，只能吃别的东西。

## CountDownLatch
为什么最后说这个类呢，因为它的功能和Golang中的WaitGroup很相似：都属于倒计时类型。  
在一些应用场景中，主线程需要等待某些条件达到要求后才能做后面的事情，同时当线程都完成后也会触发事件，以便进行后面的操作。
CountDownLatch最重要的方法是countDown()和await()，前者主要是倒数一次，后者是等待倒数到0，如果没有到达0，就阻塞等待。

CountDownLatch源码中实现了AQS的共享锁，比较简单。

## CountDownLatch VS CyclicBarrier

CountLatchDown 用于一个线程等待其他线程执行完后再执行  
CyclicBarrier 用于等待所有线程执行完后，再做操作

1.

CountDownLatch 做减法计算，count=0，唤醒阻塞线程
CyclicBarrier 做加法计算，count=屏障值（parties），唤醒阻塞线程

2.

CyclicBarrier 比 CountDownLatch 更灵活，计数达到指定值时，计数置为0重新开始，可重复利用，还有个reset方法可使用

## 多线程 join和countDownLatch.await()区别

调用thread.join() 方法必须等thread 执行完毕，当前线程才能继续往下执行，而CountDownLatch通过计数器提供了更灵活的控制，只要检测到计数器为0当前线程就可以往下执行而不用管相应的thread是否执行完毕。

## 一些陷阱
count无法归0的问题。

主任务调用一次CountDownLatch实例的await()方法时，当前线程就会一直占用一个活动线程，如果多次调用，那么就会一直占用多个活动线程，如果调用次数大于固定活动线程数，那么就可能造成阻塞队列中某些子任务一直不被执行，CountDownLatch实例的countDown()的方法一直不被调用，那么对应的主任务所在线程就会无限等待，与死锁现像一样。

# REF
> [JAVA并发辅助工具类-CountDownLatch、CyclicBarrier、Semaphore之简单介绍及和Golang的WaitGroup比较](https://studygolang.com/articles/22003)