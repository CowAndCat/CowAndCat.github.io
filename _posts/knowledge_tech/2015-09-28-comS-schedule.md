---
layout: post
title: 进程处理时间
category: commonS
comments: false
---
### 1. 概念
1、 周转时间 = 完成时间 - 到达时间  

2、平均周转时间 = 所有进程周转时间 / 进程数

3、吞吐量：单位时间内CPU完成作业的数量。

4、CPU利用率

### 2. 调度算法分类

#### 1、 先来先服务(FCFS)

先来先服务（FCFS, First Come First Serve）是最简单的调度算法，按先后顺序进行调度。

比较有利于长作业，而不利于短作业。 有利于CPU繁忙的作业，而不利于I/O繁忙的作业。

#### 2、轮转法(Round Robin)

轮转法(Round Robin)是让每个进程在就绪队列中的等待时间与享受服务的时间成正比例。

#### 3、多级反馈队列算法

多级反馈队列算法(Round Robin with Multiple Feedback)是轮转算法和优先级算法的综合和发展。

定义

设置多个就绪队列，分别赋予不同的优先级，如逐级降低，队列1的优先级最高。每个队列执行时间片的长度也不同，规定优先级越低则时间片越长，如逐级加倍。
新进程进入内存后，先投入队列1的末尾，按FCFS算法调度；若按队列1一个时间片未能执行完，则降低投入到队列2的末尾，同样按FCFS算法调度；如此下去，降低到最后的队列，则按“时间片轮转”算法调度直到完成。

仅当较高优先级的队列为空，才调度较低优先级的队列中的进程执行。如果进程执行时有新进程进入较高优先级的队列，则抢先执行新进程，并把被抢先的进程投入原队列的末尾。



优点

- 为提高系统吞吐量和缩短平均周转时间而照顾短进程。  
- 为获得较好的I/O设备利用率和缩短响应时间而照顾I/O型进程。  
- 不必估计进程的执行时间，动态调节。

#### 4、 优先级法

优先级算法（Priority Scheduling）是多级队列算法的改进，平衡各进程对响应时间的要求。适用于作业调度和进程调度，可分成抢先式和非抢先式。

#### 5、短作业优先法

短作业优先（SJF, Shortest Job First）又称为“短进程优先”SPN(Shortest Process Next)；这是对FCFS算法的改进，其目标是减少平均周转时间。
定义

对预计执行时间短的作业（进程）优先分派处理机。通常后来的短作业不抢先正在执行的作业。

#### 6、最高响应

最高响应比优先法(HRN，Highest Response_ratio Next)是对FCFS方式和SJF方式的一种综合平衡。

FCFS方式只考虑每个作业的等待时间而未考虑执行时间的长短，而SJF方式只考虑执行时间而未考虑等待时间的长短。因此，这两种调度算法在某些极端情况下会带来某些不便。

HRN调度策略同时考虑每个作业的等待时间长短和估计需要的执行时间长短，从中选出响应比最高的作业投入执行。

响应比R定义如下： R =(W+T)/T = 1+W/T

其中T为该作业估计需要的执行时间，W为作业在后备状态队列中的等待时间。

每当要进行作业调度时，系统计算每个作业的响应比，选择其中R最大者投入执行。这样，即使是长作业，随着它等待时间的增加，W / T也就随着增加，也就有机会获得调度执行。

这种算法是介于FCFS和SJF之间的一种折中算法。由于长作业也有机会投入运行，在同一时间内处理的作业数显然要少于SJF法，从而采用HRN方式时其吞吐量将小于采用SJF 法时的吞吐量。另外，由于每次调度前要计算响应比，系统开销也要相应增加。