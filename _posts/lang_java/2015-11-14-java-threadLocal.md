---
layout: post
title: java 中的ThreadLocal
category: java
comments: false
---
## 1、基本概念

### 1.1 ThreadLocal类
Java并发API为使用ThreadLocal类的局部线程变量提供了一个简洁高效的机制，

	public class ThreadLocal<T> extends Object {...}

这个类提供了一个局部线程变量。这些变量不同于其所对应的常规变量，对于常规变量，每个线程只能访问（通过get或set方法）其自身所拥有的，独立初始化变量拷贝。在一个类中，ThreadLocal类型的实例是典型的私有、静态（private static）字段，因为我们可以将其作为线程的关联状态（比如：用户ID或者事务ID）

这个类有以下方法：

- get()：返回当前线程拷贝的局部线程变量的值。
- initialValue()：返回当前线程赋予局部线程变量的初始值。
- remove()：移除当前线程赋予局部线程变量的值。
- set(T value)：为当前线程拷贝的局部线程变量设置一个特定的值。

### 1.2 怎样使用ThreadLocal？
下面的例子使用两个局部线程变量，即threadId和startDate。它们都遵循推荐的定义方法，即“private static”类型的字段。threadId用来区分当前正在运行的线程，startDate用来获取线程开启的时间。上面的信息将打印到控制台，以此验 证每一个线程管理他自己的变量拷贝。

	class DemoTask implements Runnable {
	
	   // Atomic integer containing the next thread ID to be assigned
	   private static final AtomicInteger nextId = new AtomicInteger(0);
	
	   // Thread local variable containing each thread's ID
	   private static final ThreadLocal<Integer> threadId =
	        new ThreadLocal<Integer>() {
	            @Override
	            protected Integer initialValue() {
	               return nextId.getAndIncrement();
	            }
	         };
	
	   // Returns the current thread's unique ID, assigning it if necessary
	   public int getThreadId() {
	      return threadId.get();
	   }
	
	   // Returns the current thread's starting timestamp
	   private static final ThreadLocal<Date> startDate =
	       new ThreadLocal<Date>() {
	           protected Date initialValue() {
	               return new Date();
	           }
	       };
	
	   @Override
	   public void run() {
	      System.out.printf("Starting Thread: %s : %sn",
	                        getThreadId(), startDate.get());
	      try {
	         TimeUnit.SECONDS.sleep((int) Math.rint(Math.random() * 10));
	      } catch (InterruptedException e) {
	         e.printStackTrace();
	      }
	      System.out.printf("Thread Finished: %s : %sn",
	                        getThreadId(), startDate.get());
	   }
	}

现在要验证变量本质上能够维持其自身状态，而与多线程的多次初始化无关。我们首先需要创建执行这个任务的三个线程，然后开启线程，接着验证它们打印到控制台中的信息。

	Starting Thread: 0 : Wed Dec 24 15:04:40 IST 2014
	Thread Finished: 0 : Wed Dec 24 15:04:40 IST 2014
	
	Starting Thread: 1 : Wed Dec 24 15:04:42 IST 2014
	Thread Finished: 1 : Wed Dec 24 15:04:42 IST 2014
	
	Starting Thread: 2 : Wed Dec 24 15:04:44 IST 2014
	Thread Finished: 2 : Wed Dec 24 15:04:44 IST 2014

在上面的输出中，打印出的声明序列每次都在变化。我已经把它们放到了序列中，这样对于每一个线程实例，我们都可以清楚地辨别出，局部线程变量保持着安全状态，而绝不会混淆。自己尝试下！

**局部线程通常使用在这样的情况下，当你有一些对象并不满足线程安全，但是你想避免在使用synchronized关键字、块时产生的同步访问，那么，让每个线程拥有它自己的对象实例。**

注意：局部变量是同步或局部线程的一个好的替代，它总是能够保证线程安全。唯一可能限制你这样做的是你的应用设计约束。

警告：在webapp服务器上，可能会保持一个线程池，那么ThreadLocal变量会在响应客户端之前被移除，因为当前线程可能被下一个请求重复使用。而 且，如果在使用完毕后不进行清理，它所保持的任何一个对类的引用—这个类会作为部署应用的一部分加载进来—将保留在永久堆栈中，永远不会被垃圾回收机制回收。

## 2、 使用ThreadLocal变量的时机和方法

使用时机：  

举个例子，想象你在开发一个电子商务应用，你需要为每一个控制器处理的顾客请求，生成一个唯一的事务ID，同时将其传到管理器或DAO的业务方法中，以便记录日志。一种方案是将事务ID作为一个参数，传到所有的业务方法中。但这并不是一个好的方案，它会使代码变得冗余。

你可以使用ThreadLocal类型的变量解决这个问题。首先在控制器或者任意一个预处理器拦截器中生成一个事务ID，然后在ThreadLocal中 设置事务ID，最后，不论这个控制器调用什么方法，都能从threadlocal中获取事务ID。而且这个应用的控制器可以同时处理多个请求，同时在框架 层面，因为每一个请求都是在一个单独的线程中处理的，所以事务ID对于每一个线程都是唯一的，而且可以从所有线程的执行路径获取。

## 3、使用局部变量实现线程同步

Bank.java代码如下：

	public class Bank {  
	
	    private static ThreadLocal<Integer> count = new ThreadLocal<Integer>(){  
	
	        @Override  
	        protected Integer initialValue() {  
	            // TODO Auto-generated method stub  
	            return 0;  
	        }  
	
	    };  
	
	    // 存钱  
	    public void addMoney(int money) {  
	        count.set(count.get()+money);  
	        System.out.println(System.currentTimeMillis() + "存进：" + money);  
	
	    }  
	
	    // 取钱  
	    public void subMoney(int money) {  
	        if (count.get() - money < 0) {  
	            System.out.println("余额不足");  
	            return;  
	        }  
	        count.set(count.get()- money);  
	        System.out.println(+System.currentTimeMillis() + "取出：" + money);  
	    }  
	
	    // 查询  
	    public void lookMoney() {  
	        System.out.println("账户余额：" + count.get());  
	    }  
	}

运行效果：

	余额不足  
	账户余额：0  
	
	余额不足  
	账户余额：0  
	
	1441794247939存进：100  
	账户余额：100  
	
	余额不足  
	1441794248940存进：100  
	账户余额：0  
	
	账户余额：200  
	
	余额不足  
	账户余额：0  
	
	1441794249941存进：100  
	账户余额：300
	
看了运行效果，一开始一头雾水，怎么只让存，不让取啊？看看ThreadLocal的原理：

**如果使用ThreadLocal管理变量，则每一个使用该变量的线程都获得该变量的副本，副本之间相互独立，这样每一个线程都可以随意修改自己的变量副本，而不会对其他线程产生影响。**现在明白了吧，原来每个线程运行的都是一个副本，也就是说存钱和取钱是两个账户，知识名字相同而已。所以就会发生上面的效果。

ThreadLocal与同步机制

a.ThreadLocal与同步机制都是为了解决多线程中相同变量的访问冲突问题  
b.前者采用以”空间换时间”的方法，后者采用以”时间换空间”的方式