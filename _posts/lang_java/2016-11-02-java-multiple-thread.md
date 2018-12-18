---
layout: post
title: Java中多线程
category: java
comments: false
---
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


