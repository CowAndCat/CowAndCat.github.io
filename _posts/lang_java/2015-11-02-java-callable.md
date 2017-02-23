---
layout: post
title: Java Callable
category: java
comments: false
---
Callable接口则提供了一种有返回值的多线程实现方法。

## 1、Callable 和 Runnable的区别
Callable 和 Runnable 的使用方法大同小异， 区别在于：

1. Callable 使用 **call（）** 方法， Runnable 使用 **run()** 方法
2. call() 可以**返回值**， 而 run()方法不能返回。
3. call() 可以**抛出受检查的异常**，比如ClassNotFoundException， 而run()不能抛出受检查的异常。
4. ExecutorService 在Callable中使用的是submit()， 在Runnable中使用的是 execute()。

## 2、Callable和Future
Callable和Future，它俩很有意思的，一个产生结果，一个拿到结果。

运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。

通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。

submit()方法回产生Future对象，它用Callable返回结果的特定类型进行了参数化。可以用isDone()方法来查询Future是否已经完成，当任务完成时，它具有一个结果，可以调用get()方法获取该结果。也可以不用isDone()进行检查就直接调用get()，在这种情况下，get()将阻塞，直至结果准备就绪。还可以在试图调用get()来获取结果之前，先调用具有超时的get()，或者调用isDone()来查看任务是否完成。

### 3、java 创建线程的三种方法Callable,Runnable，Thread比较及用法

#### 3.1 实现Runnable接口来创建Thread线程：

	class SomeRunnable implements Runnable{
		public void run(){
			//do something here
		}
	}

	//调用
	Thread someThread = new Thread(new SomeRunnable());
	someThread.start();

方法执行完，线程就消亡了。

#### 3.2实现Callable接口来创建Thread线程

	public interface Callable<V>   { 
		V call（） throws Exception;  
	}

	class SomeCallable<Integer> implements Callable<Integer>{
		public Integer call() throws Exception{
			//do something here
			//return new Random().nextInt(100);
		}
	}

	//调用
	FutureTask<Integer> future = new FutureTask<Integer>(callable);  
    new Thread(future).start();  
	future.get()；

#### 3.3 继承Thread类来创建

	class SomeThread extends Thread{
		public void run(){
			//do something here
		}
	}

	//调用
	Thread someThread = new SomeThread();
	someThread.start();
