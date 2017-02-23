---
layout: post
title: guava google的扩展类库
category: java
comments: false
---

# 一、Guava介绍

Guava是一个 Google 的基于java1.6的类库集合的扩展项目，包括 collections, caching, primitives support, concurrency libraries, common annotations, string processing, I/O, 等等. 这些高质量的 API 以使你的JAVa代码更加优雅，更加简洁，让你工作更加轻松愉悦。

## 源码包的简单说明： 

	com.google.common.annotations：普通注解类型。  
	com.google.common.base：基本工具类库和接口。     
	com.google.common.cache：缓存工具包，非常简单易用且功能强大的JVM内缓存。   
	com.google.common.collect：带泛型的集合接口扩展和实现，以及工具类，这里你会发现很多好玩的集合。   
	com.google.common.eventbus：发布订阅风格的事件总线。   
	com.google.common.hash： 哈希工具包。   
	com.google.common.io：I/O工具包。   
	com.google.common.math：原始算术类型和超大数的运算工具包。   
	com.google.common.net：网络工具包。   
	com.google.common.primitives：八种原始类型和无符号类型的静态工具包。   
	com.google.common.reflect：反射工具包。   
	com.google.common.util.concurrent：多线程工具包。  

# 二、使用手册

## 1.  基本工具类：让使用Java语言更令人愉悦。

1. 使用和避免 null：null 有语言歧义， 会产生令人费解的错误， 反正它总是让人不爽。很多 Guava 的工具类在遇到 null 时会直接拒绝或出错，而不是默默地接受他们。
2. 前提条件：让方法中的条件检查更简单
3. 常见的对象方法： 简化了Object常用方法的实现， 如 hashCode() 和 toString()。
4. 排序： Guava 强大的 "fluent Comparator"比较器， 提供多关键字排序。
5. Throwable类： 简化了异常检查和错误传播。

## 2.  集合类：集合类库是 Guava 对 JDK 集合类的扩展。

1. Immutable collections（不变的集合）： 防御性编程， 不可修改的集合，并且提高了效率。
2. New collection types(新集合类型)：JDK collections 没有的一些集合类型，主要有：multisets，multimaps，tables， bidirectional maps等等
3. Powerful collection utilities（强大的集合工具类）： java.util.Collections 中未包含的常用操作工具类
4. Extension utilities（扩展工具类）: 给 Collection 对象添加一个装饰器，或实现迭代器

## 3.  缓存
本地缓存，可以很方便的操作缓存对象，并且支持各种缓存失效行为模式。

## 4.  Functional idioms（函数式）
 简洁, Guava实现了Java的函数式编程，可以显著简化代码。

## 5. Concurrency（并发）
强大,简单的抽象,让我们更容易实现简单正确的并发性代码。

　　1. ListenableFuture（可监听的Future）: Futures,用于异步完成的回调。
　　2. Service: 控制事件的启动和关闭，为你管理复杂的状态逻辑。

## 6. Strings
一个非常非常有用的字符串工具类: 提供 splitting，joining， padding 等操作。

## 7. Primitives
扩展 JDK 中未提供的对原生类型（如int、char等）的操作， 包括某些类型的无符号的变量。

## 8. Ranges
 Guava 一个强大的 API，提供 Comparable 类型的范围处理， 包括连续和离散的情况。

## 9. I/O
 简化 I/O 操作, 特别是对 I/O 流和文件的操作, for Java 5 and 6.

## 10. Hashing
 提供比 Object.hashCode() 更复杂的 hash 方法, 提供 Bloom filters.

## 11. EventBus
 基于发布-订阅模式的组件通信，但是不需要明确地注册在委托对象中。

## 12. Math
 优化的 math 工具类，经过完整测试。

## 13. Reflection
 Guava 的 Java 反射机制工具类。

# 三、ListenableFuture 解析

Guava 定义了 ListenableFuture接口并继承了JDK concurrent包下的Future 接口。

强烈地建议在代码中多使用ListenableFuture来代替JDK的 Future

## 1. 接口
传统JDK中的Future通过异步的方式计算返回结果:在多线程运算中可能在没有结束返回结果，Future是运行中的多线程的一个引用句柄，确保在服务执行返回一个Result。

ListenableFuture可以允许你注册回调方法(callbacks)，在运算（多线程执行）完成的时候进行调用,  或者在运算（多线程执行）完成后立即执行。这样简单的改进，使得可以明显的支持更多的操作，这样的功能在JDK concurrent中的Future是不支持的。

ListenableFuture 中的基础方法是**addListener(Runnable, Executor)**, 该方法会在多线程运算完的时候，指定的Runnable参数传入的对象会被指定的Executor执行


## 2. ListenableFuture的创建
对应JDK中的 <u>ExecutorService.submit(Callable) </u>提交多线程异步运算的方式，Guava 提供了<u>ListeningExecutorService</u> 接口, 该接口返回 <u>ListenableFuture</u> 而相应的 ExecutorService 返回普通的 Future。将 ExecutorService 转为 <u>ListeningExecutorService</u>，可以使用<u>MoreExecutors.listeningDecorator(ExecutorService)</u>进行装饰。

	ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));
	ListenableFuture explosion = service.submit(new Callable() {
	  public Explosion call() {
	    return pushBigRedButton();
	  }
	});
	Futures.addCallback(explosion, new FutureCallback() {
	  // we want this handler to run immediately after we push the big red button!
	  public void onSuccess(Explosion explosion) {
	    walkAwayFrom(explosion);
	  }
	  public void onFailure(Throwable thrown) {
	    battleArchNemesis(); // escaped the explosion!
	  }
	});

## 3. application
使用ListenableFuture 最重要的理由是它可以进行一系列的复杂链式的异步操作。

	ListenableFuture rowKeyFuture = indexService.lookUp(query);
	AsyncFunction<RowKey, QueryResult> queryFunction =
		new AsyncFunction<RowKey, QueryResult>() {
			public ListenableFuture apply(RowKey rowKey) {
				return dataService.read(rowKey);
			}
		};
	ListenableFuture queryFuture = Futures.transform(rowKeyFuture, queryFunction, queryExecutor);

不同的操作可以在不同的Executors中执行，单独的ListenableFuture 可以有多个操作等待。

当一个操作开始的时候其他的一些操作也会尽快开始执行–“fan-out”–ListenableFuture 能够满足这样的场景：促发所有的回调（callbacks）。

反之更简单的工作是，同样可以满足“fan-in”(扇入，指应用程序模块之间的层次调用情况，设计良好的软件结构，通常顶层扇出比较大，中间扇出小，底层模块则有大扇入。）场景，促发ListenableFuture 获取（get）计算结果，同时其它的Futures也会尽快执行：可以参考 [the implementation of Futures.allAsList](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/src-html/com/google/common/util/concurrent/Futures.html#line.1276) 。

# 四、参考
>[Google Guava官方教程（中文版）](http://ifeve.com/google-guava/)

>[http://ifeve.com/google-guava-listenablefuture/](http://ifeve.com/google-guava-listenablefuture/)