---
layout: post
title: Java中的频次控制：RateLimiter
category: java
comments: false
---

## 一、RateLimiter是什么？
RateLimiter是guava提供的基于令牌桶算法的实现类，可以非常简单的完成限流特技，并且根据系统的实际情况来调整生成token的速率。

通常可应用于抢购限流防止冲垮系统；限制某接口、服务单位时间内的访问量，譬如一些第三方服务会对用户访问量进行限制；限制网速，单位时间内只允许上传下载多少字节等。

RateLimiter经常用于限制对一些物理资源或者逻辑资源的访问速率。与Semaphore 相比，Semaphore 限制了并发访问的数量而不是使用速率。

## 二、如何使用
Code first:

    import com.google.common.util.concurrent.RateLimiter;  
    public class Demo1 {  
        public static void main(String[] args) {  
            //0.5代表一秒最多多少个  
            RateLimiter rateLimiter = RateLimiter.create(0.5);  
            List<Runnable> tasks = new ArrayList<Runnable>();  
            for (int i = 0; i < 10; i++) {  
                tasks.add(new UserRequest(i));  
            }  
            ExecutorService threadPool = Executors.newCachedThreadPool();  
            for (Runnable runnable : tasks) {  
                System.out.println("等待时间：" + rateLimiter.acquire());  
                threadPool.execute(runnable);  
            }  
        }  
    }

RateLimiter.create(0.5) 创建一个2s执行一次的限流器。  
rateLimiter.acquire()该方法会阻塞线程，直到令牌桶中能取到令牌为止才继续向下执行，并返回等待的时间。

上述代码适用于 有很多任务，但希望每秒不超过N个 的场景。（用N替换0.5）

另外一个例子，想象下我们制造了一个数据流，并希望以每秒5kb的速率处理它。可以通过要求每个字节代表一个许可，然后指定每秒5000个许可来完成：

    // 每秒5000b
    final RateLimiter rateLimiter = RateLimiter.create(5000.0); 

    void submitPacket(byte[] packet) {
        rateLimiter.acquire(packet.length);
        networkService.send(packet);
    }

有一点很重要，那就是请求的许可数从来不会影响到请求本身的限制（调用acquire(1) 和调用acquire(1000) 将得到相同的限制效果，如果存在这样的调用的话），但会影响下一次请求的限制，也就是说，如果一个高开销的任务抵达一个空闲的RateLimiter，它会被马上许可，但是下一个请求会经历额外的限制，从而来偿付高开销任务。

注意：RateLimiter 并不提供公平性的保证。

## 三、相关函数

    public static RateLimiter create(double permitsPerSecond)   // 每秒的令牌数(或叫QPS)
    public static RateLimiter create(double permitsPerSecond, long warmupPeriod, TimeUnit unit) // warmupPeriod

    public final void setRate(double permitsPerSecond)

    public double acquire()
    public double acquire(int permits)
    public boolean tryAcquire()
    public boolean tryAcquire(int permits)
    public boolean tryAcquire(long timeout, TimeUnit unit)
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)

## REF 
>[使用RateLimiter完成简单的大流量限流，抢购秒杀限流](http://www.cnblogs.com/yeyinfu/p/7316972.html)  
>[RateLimiter](http://xiaobaoqiu.github.io/blog/2015/07/02/ratelimiter/)  
>[Guava官方文档-RateLimiter类](https://www.cnblogs.com/exceptioneye/p/4824394.html)