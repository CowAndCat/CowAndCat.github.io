---
layout: post
title: Java 定时任务实现原理
category: java
comments: false
---

## 一、Java定时任务

在jdk自带的库中,有两种技术可以实现定时任务。一种是使用Timer,另外一个则是ScheduledThreadPoolExecutor。

### 1.1 Timer和ScheduledThreadPoolExecutor的区别

1. 由于Timer是单线程的,如果一次执行多个定时任务，会导致某些任务被其他任务所阻塞。比如A任务每秒执行一次，B任务10秒执行一次，但是一次执行5秒，就会导致A任务在长达5秒都不会得到执行机会。  
而ScheduledThreadPoolExecutor是基于线程池的，可以动态的调整线程的数量，所以不会有这个问题。

2. 如果执行多个任务，在Timer中一个任务的崩溃会导致所有任务崩溃，从而所有任务都停止执行。  
而ScheduledThreadPoolExecutor则不会。

3. Timer的执行周期时间依赖于系统时间，timer中，获取到堆顶任务执行时间后，如果执行时间还没到，会计算出需要休眠的时间=(执行时间-系统时间),如果系统时间被调整，就会导致休眠时间无限拉长，后面就算改回来了任务也因为在休眠中而得不到执行的机会。  
ScheduledThreadPoolExecutor由于用是了nanoTime来计算执行周期的,所以和系统时间是无关的,无论系统时间怎么调整都不会影响到任务调度。

注意的是,**nanoTime和系统时间是完全无关的(之前一直以为只是时间戳的纳秒级粒度)**,关于nanoTime的介绍如下:

> nanoTime返回最准确的可用系统计时器的当前值，以毫微秒为单位。  
> **此方法只能用于测量已过的时间，与系统或钟表时间的其他任何时间概念无关**。返回值表示从某一固定但任意的时间算起的毫微秒数（或许从以后算起，所以该值可能为负）。此方法提供毫微秒的精度，但不是必要的毫微秒的准确度。它对于值的更改频率没有作出保证。其取值范围的上限约 292 年（2^^63 毫微秒）。所以它很少用来计算当前日期，通常都是测量。

总体来说,Timer除了在版本兼容性上面略胜一筹以外(Timer是jdk1.3就支持的，而ScheduledThreadPoolExecutor在jdk1.5才出现)，其余全部被ScheduledThreadPoolExecutor碾压。所以日常技术选型中，也**推荐使用ScheduledThreadPoolExecutor来实现定时任务**。

## 二、Timer的实现原理

### 2.1 Timer的使用

    class MyTask extends TimerTask {
        @Override
        public void run() {
            System.out.println("hello world");
        }
    }
    public class TimerDemo {
        public static void main(String[] args) {
            //创建定时器对象
            Timer t = new Timer();
            //在3秒后执行MyTask类中的run方法,后面每10秒跑一次
            t.schedule(new MyTask(), 3000,10000);

        }
    }

### 2.2 深入源码

Timer类：

    //TaskQueue是存放定时任务的队列
    //这个TaskQueue 也是Timer内部自定义的一个队列,这个队列通过最小堆来维护队列
    //下一次执行时间距离现在最小的会被放在堆顶，到时执行线程直接获取堆顶任务并判断是否执行即可
    private final TaskQueue queue = new TaskQueue();
    //负责执行定时任务的线程
    private final TimerThread thread = new TimerThread(queue);
    public Timer() {
            this("Timer-" + serialNumber());
    }
    public Timer(String name) {
            //设置线程的名字,并且启动这个线程
            thread.setName(name);
            thread.start();
    }

总结一下：用最小堆来存储需要调度的任务，由一个线程来执行调度。整个过程类似生产者和消费者模式。

TimerThread类继承了Thread类，可以直接当线程使唤：

    //在Timer中初始化的时候会将Timer的Queue赋值进来
    TimerThread(TaskQueue queue) {
            this.queue = queue;
    }
    public void run() {
        try {
            //进入自旋,开始不断的从任务队列中获取定时任务来执行
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }
    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                //加同步
                synchronized(queue) {
                    //如果任务队列为空,并且newTasksMayBeScheduled为true,就休眠等待,直到有任务进来就会唤醒这个线程
                    //如果有人调用timer的cancel方法，newTasksMayBeScheduled会变成false
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait();
                    if (queue.isEmpty())
                        break; 

                    // 获取当前时间和下次任务执行时间
                    long currentTime, executionTime;
                    //获取队列中最早要执行的任务
                    task = queue.getMin();
                    synchronized(task.lock) {
                        //如果这个任务已经被结束了，就从队列中移除
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue;  // No action required, poll queue again
                        }
                        //获取当前时间和下次任务执行时间
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        //判断任务执行时间是否小于当前时间,表示小于，就说明可以执行了
                        if (taskFired = (executionTime<=currentTime)) {
                            //如果任务的执行周期是0,说明只要执行一次就好了,就从队列中移除它,这样下一次就不会获取到该任务了
                            if (task.period == 0) {
                                queue.removeMin();
                                task.state = TimerTask.EXECUTED;
                            } else { 
                                //重新设置该任务下一次的执行时间
                                //如果之前设置的period小于0,就用当前时间-period,等于就是当前时间加上周期值
                                //这里的下次执行时间就是当前的执行时间加上周期值
                                //这里涉及到是否以固定频率调用任务的问题,下面再详细讲解
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period
                                                : executionTime + task.period);
                            }
                        }
                    }
                    //如果任务的执行时间还没到，就计算出还有多久才到达执行时间,然后线程进入休眠
                    if (!taskFired) 
                        queue.wait(executionTime - currentTime);
                }
                //如果任务的执行时间到了,就执行这个任务
                if (taskFired)
                    task.run();
            } catch(InterruptedException e) {
            }
        }
    }

回到2.1中的schedule方法，源码如下：

    //Timer.java
    public void schedule(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        //调用内部的一个方法
        sched(task, System.currentTimeMillis()+delay, -period);
    }

    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        // 如果设定的定时任务周期太长,就将其除以2
        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        //加锁同步
        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");
            //设置任务的各个属性
            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }
            //将任务加入到队列中
            queue.add(task);
            //如果任务加入队列后排在堆顶，说明该任务可能马上可以执行了,那就唤醒执行线程
            if (queue.getMin() == task)
                queue.notify();
        }
    }           

### 2.3 总结
**Timer的原理比较简单,当我们初始化Timer的时候,timer内部会启动一个线程，并且初始化一个优先级队列，该优先级队列使用了最小堆的技术来将最早执行时间的任务放在堆顶。**

当我们调用schedule方法的时候,其实就是生成一个任务然后插入到该优先级队列中。

最后，timer内部的线程会从优先级队列的堆顶获取任务，获取到任务后，先判断执行时间是否到了，如果到了先设置下一次的执行时间并调整堆，然后执行任务。如果没到执行时间那线程就休眠一段时间。 

关于计算下次任务执行时间的策略，会根据传入peroid的值来判断使用哪种策略： 

- 如果peroid是负数,那下一次的执行时间就是当前时间+peroid的值。
- 如果peroid是正数，那下一次执行时间就是该任务这次的执行时间+peroid的值。 （固定频率）

这两个策略的不同点在于，如果计算下次执行时间是以当前时间为基数，那它就不是以固定频率来执行任务的。因为**Timer是单线程执行任务的**,如果A任务执行周期是10秒，但是有个B任务执行了20几秒，那么下一次A任务的执行时间就要等B执行完后轮到自己时，再过10秒才会执行下一次。 
如果策略是这次任务的执行时间+peroid的值就是按固定频率不断执行任务了。


## 三、ScheduledThreadPoolExecutor的实现原理

### 3.1 使用

    ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(8);
    scheduledExecutorService.scheduleAtFixedRate(
        new Runnable() {
            @Override
            public void run() {
                System.out.println("hello world");
            }
        }, 1, 3, TimeUnit.SECONDS);

### 3.2 深入源码
ScheduledThreadPoolExecutor是基于线程池实现调度的。

    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        //将任务进行一层封装,最后得到一个ScheduledFutureTask对象
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        //进行一些装饰,其实就是返回sft这个对象
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //提交给线程池执行
        delayedExecute(t);
        return t;
    }
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        //如果线程池已经关闭,就拒绝这个任务
        if (isShutdown())
            reject(task);
        else {
            //将当前任务加入到任务队列中去
            super.getQueue().add(task);
            //判断线程池是否关闭了,然后判断是否需要移除这个任务
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))
                task.cancel(false);
            else
                //因为这里的定时任务是直接放到任务队列中,所以需要保证已经有worker启动了
                ensurePrestart();
        }
    }
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        //如果worker的数量小于corePoolSize,那就启动一个worker,用来消费任务队列的任务
        if (wc < corePoolSize)
            addWorker(null, true);
        //worker的数量为0也直接启动一个worker
        else if (wc == 0)
            addWorker(null, false);
    }

我们提交的任务被封装成一个ScheduledFutureTask然后提交给任务队列，同时如果发现worker的数量少于设置的corePoolSize，我们还会启动一个worker线程。 

但是，Q1:我们怎么保证worker不会马上就从任务队列中获取任务然后直接执行呢？(这样我们设定的延迟执行就没有效果了)
另外，Q2:怎么保证任务执行完下一次在一定周期后还会再执行呢，也就是怎么保证任务的延迟执行和周期执行？ 

我们先来看一下**任务的延迟执行的解决方案。其实就是修改任务队列的实现，通过将任务队列变成延迟队列**，worker不会马上获取到任务队列中的任务了。只有任务的时间到了，worker线程才能从延迟队列中获取到任务并执行。 

在ScheduledThreadPoolExecutor中，定义了**DelayedWorkQueue类来实现延迟队列**。DelayedWorkQueue内部使用了最小堆的数据结构，当任务插入到队列中时，会根据执行的时间自动调整在堆中的位置，最后执行时间最近的那个会放在堆顶。 

当worker要去队列获取任务时，如果堆顶的执行时间还没到，那么worker就会阻塞一定时间后才能获取到那个任务，这样就实现了任务的延迟执行。(Q1解决)

解决了任务的延迟执行问题，接下来就是任务的周期执行的解决方案了。周期执行和前面封装的ScheduledFutureTask有关。

ScheduledFutureTask的run方法：

    public void run() {
        //先判断任务是否周期执行
        boolean periodic = isPeriodic();
        //判断是否能执行任务
        if (!canRunInCurrentRunState(periodic))
            cancel(false);
        //判断是否周期性任务
        else if (!periodic)
            //不是的话执行执行run方法
            ScheduledFutureTask.super.run();
        else if (ScheduledFutureTask.super.runAndReset()) {
            //如果是周期性任务,那就设置下一次的执行时间
            setNextRunTime();
            //重新将任务放到队列中,然后等待下一次执行
            reExecutePeriodic(outerTask);
        }
    }
    private void setNextRunTime() {
        //根据peroid的正负来判断下一次执行时间的计算策略
        //和timer的下一次执行时间计算策略有点像
        long p = period;
        if (p > 0)
            time += p;
        else
            time = triggerTime(-p);
    }
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        //先判断是否可以在当前状态下执行
        if (canRunInCurrentRunState(true)) {
            //重新加任务放到任务队列中
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }

从源码可以看出,当任务执行完后，**如果该任务是周期性任务，那么会重新计算下一次执行时间，然后重新放到任务队列中等待下一次执行**。（解决Q2）

### 3.3 总结
**ScheduledThreadPoolExecutor的实现是基于java线程池。通过对任务进行一层封装来实现任务的周期执行，以及将任务队列改成延迟队列来实现任务的延迟执行。**

我们将任务放入任务队列的同时，会尝试开启一个worker来执行这个任务(如果当前worker的数量小于corePoolSize)。由于这个任务队列是一个延迟队列，只有任务执行时间达到才能获取到任务，因此worker只能阻塞等到有队列中有任务到达才能获取到任务执行。

当任务执行完后，会检查自己是否是一个周期性执行的任务。如果是的话，就会重新计算下一次执行的时间，然后重新将自己放入任务队列中。

关于下一次任务的执行时间的计算规则，和Timer差不多。

## REF
> [Java 定时任务实现原理详解](https://blog.csdn.net/u013332124/article/details/79603943)  
> [System.nanoTime与System.currentTimeMillis的区别](https://www.cnblogs.com/langtianya/p/4968465.html)