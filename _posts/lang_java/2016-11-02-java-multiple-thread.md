---
layout: post
title: Java中多线程
category: java
comments: false
---
面试准备：JDK 自带的四种线程池、ThreadPoolExecutor 类中的最重要的构造器里面的七个参数，再讲了下线程任务进入线程池和核心线程数、缓冲队列、最大线程数量比较。

## 1.线程池
>Java线程池使用说明  
>http://www.oschina.net/question/565065_86540

线程池作用就是限制系统中执行线程的数量。

线程的使用在java中占有极其重要的地位，在jdk1.4极其之前的jdk版本中，关于线程池的使用是极其简陋的。在jdk1.5之后这一情况有了很大的改观。Jdk1.5之后加入了java.util.concurrent包，这个包中主要介绍java中线程以及线程池的使用。为我们在开发中处理线程的问题提供了非常大的帮助。

根据系统的环境情况，可以自动或手动设置线程数量，达到运行的最佳效果；少了浪费了系统资源，多了造成系统拥挤效率不高。用线程池控制线程数量，其他线程排队等候。一个任务执行完毕，再从队列的中取最前面的任务开始执行。若队列中没有等待进程，线程池的这一资源处于等待。当一个新任务需要运行时，如果线程池中有等待的工作线程，就可以开始运行了；否则进入等待队列。

### 1.1 为什么要用线程池  

1. 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。
2. 可以根据系统的承受能力，调整线程池中工作线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)。

## 2. 调度器 Executor
Java里面线程池的顶级接口是Executor，但是严格意义上讲Executor并不是一个线程池，而只是一个执行线程的工具。真正的线程池接口是ExecutorService。

Executors可以创建的三种(JAVA8增加了一种，共四种)线程池的特点及适用范围。

### 2.1 public static ExecutorService newCachedThreadPool() 
创建一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们。对于执行很多短期异步任务的程序而言，这些线程池通常可提高程序性能。

如果线程池的大小超过了处理任务所需要的线程，
那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。

### 2.2 public static ExecutorService newFixedThreadPool(int nThreads)
创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程。

线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

### 2.3 public static ExecutorService newSingleThreadExecutor()
创建一个使用单个 worker 线程的 Executor，以无界队列方式来运行该线程。

这个线程池只有一个线程在工作，也就是相当于单线程串行执行所有任务。如果这个唯一的线程因为异常结束，那么会有一个新的线程来替代它。此线程池保证所有任务的执行顺序按照任务的提交顺序执行。

### 2.4 newScheduledThreadPool()
创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

-----
这些方法都可以配合接口ThreadFactory的实例一起使用。并且返回一个ExecutorService接口的实例。

```
package com.zj.concurrency.executors;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
 
public class FixedThreadPool {
    public static void main(String[] args) {
       ExecutorService exec = Executors.newFixedThreadPool(2);
       for (int i = 0; i < 5; i++)
           exec.execute(new MyThread(i));
       exec.shutdown(); //可以控制关闭
    }
}
```

## 3. 线程池中ThreadPoolExecutor构造器参数介绍

### 3.1 参数介绍
线程池中ThreadPoolExecutor构造器有7个参数:

    public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
      //...
    }

- 1.corePoolSize
    核心池大小corePoolSize：表示线程池维护线程的最少数量
- 2.maximumPoolSize  
    最大池大小maximumPoolSize：表示线程池维护线程的最大数量
- 3.workQueue  
    阻塞队列workQueue：表示如果任务数量超过核心池大小，多余的任务添加到阻塞队列中
- 3.1 corePoolSize、workQueue、maximumPoolSize的关系  
    前提假设：向线程池每添加一个任务就sleep。也就是说假设任务数与线程数一一对应，每添加一个任务就对应的创建一个线程，并且一直等待其他线程。 
    因为可能某一个线程执行了两个任务，看不出效果。   
    注意，线程池并没有标记哪个线程是核心线程，哪个是非核心线程，线程池只关心核心线程的数量。  
    a.

        if 任务数 <= 核心池大小 
            则每添加一个任务就会创建一个线程来执行该任务,线程最大数量等于核心池大小

    b.

        if 任务数 > 核心池大小 && 任务数 <= 核心池大小 + 阻塞队列大小
            则线程数量等于核心池大小，其余任务放入到阻塞队列中

    c.

        if 任务数 > 核心池大小 + 阻塞队列大小 && 任务数 <= 最大池大小
            则会创建新的线程来处理新的任务

    d.

        if 任务数 > 最大池大小
            则会采用拒绝策略handler

- 4.参数keepAliveTime
    表示空闲线程的存活时间。
- 5.参数unit
    表示keepAliveTime的单位。
- 6.参数threadFactory
    指定创建线程的工厂，例如：Executors.defaultThreadFactory()
- 7.参数handler
    线程池对拒绝任务的处理策略（表示当workQueue已满，且池中的线程数达到maximumPoolSize时，线程池拒绝添加新任务时采取的策略。）

在线程池中常用的阻塞队列有以下2种：

- （1）SynchronousQueue<Runnable>：此队列中不缓存任何一个任务。向线程池提交任务时，如果没有空闲线程来运行任务，则入列操作会阻塞。当有线程来获取任务时，出列操作会唤醒执行入列操作的线程。从这个特性来看，SynchronousQueue是一个无界队列，因此当使用SynchronousQueue作为线程池的阻塞队列时，参数maximumPoolSizes没有任何作用。（生产者和消费者互相等待对方，握手，然后一起离开。）
- （2）LinkedBlockingQueue<Runnable>：顾名思义是用链表实现的队列，可以是有界的，也可以是无界的，但在Executors中默认使用无界的。

handler一般可以采取以下四种取值：

|取值|解释|
|--|--|
|ThreadPoolExecutor.AbortPolicy()|抛出RejectedExecutionException异常
|ThreadPoolExecutor.CallerRunsPolicy()|由向线程池提交任务的线程来执行该任务
|ThreadPoolExecutor.DiscardOldestPolicy()|抛弃最旧的任务（最先提交而没有得到执行的任务）|
|ThreadPoolExecutor.DiscardPolicy()|抛弃当前的任务|

### 3.2 其它有关涉及池中线程数量的相关方法

    public void allowCoreThreadTimeOut(boolean value)
    public int prestartAllCoreThreads()

默认情况下，当池中有空闲线程，且线程的数量大于corePoolSize时，空闲时间超过keepAliveTime的线程会自行销毁，池中仅仅会保留corePoolSize个线程。如果线程池中调用了allowCoreThreadTimeOut这个方法，则空闲时间超过keepAliveTime的线程全部都会自行销毁，而不必理会corePoolSize这个参数。

如果池中的线程数量小于corePoolSize时，调用prestartAllCoreThreads方法，则无论是否有待执行的任务，线程池都会创建新的线程，直到池中线程数量达到corePoolSize。

### 3.3 ThreadPoolExecutor和四个创建线程池函数的关系。

为了防止使用者错误搭配ThreadPoolExecutor构造函数的各个参数，以及更加方便简洁的创建ThreadPoolExecutor对象，JavaSE中又定义了Executors类，Eexcutors类提供了创建常用配置线程池的方法。

newCachedThreadPool：使用SynchronousQueue作为阻塞队列，队列无界，线程的空闲时限为60秒。这种类型的线程池非常适用IO密集的服务，因为IO请求具有密集、数量巨大、不持续、服务器端CPU等待IO响应时间长的特点。

newFixedThreadPool：需指定核心线程数，核心线程数和最大线程数相同，使用LinkedBlockingQueue 作为阻塞队列，队列无界，线程空闲时间0秒。

newSingleThreadExecutor：池中只有一个线程工作，阻塞队列无界，它能保证按照任务提交的顺序来执行任务。

newScheduledThreadPool：创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求。

## REF
>[线程池中ThreadPoolExecutor构造器参数介绍](https://blog.csdn.net/xiangliqu/article/details/78223104)

