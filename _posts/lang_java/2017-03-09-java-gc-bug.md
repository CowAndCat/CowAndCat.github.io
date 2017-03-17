---
layout: post
title: Java 查找Full gc的起因
category: java
comments: false
---
最近遇到一个频繁gc的服务，记录一下查找问题的过程

## 1. 先用jps 和 jstack定位pid

推荐用如下命令代：

	ps aux|grep java

---

	[running]mqq@10.242.50.116:~$ jps
	18648 Main
	19097 Main
	17075 Main
	20069 Jps
	28296 Main
	18110 Main
	10540 Main

然后根据线程名字（自己找特征）定位查找：

	[running]mqq@10.242.50.116:~$ jstack 28296 |grep 'UserBase'
	"taserverThreadPool-exec-null.UserBase.YuewenRiskControlServer.YuewenRiskControlServant-25" prio=10 tid=0x00007f6910005800 nid=0x6f77 waiting on condition [0x00007f68795d4000]
	"taserverThreadPool-exec-null.UserBase.YuewenRiskControlServer.YuewenRiskControlServant-24" prio=10 tid=0x00007f6918005800 nid=0x6f76 waiting on condition [0x00007f68796d5000]
	"taserverThreadPool-exec-null.UserBase.YuewenRiskControlServer.YuewenRiskControlServant-23" prio=10 tid=0x00007f691c005800 nid=0x6f7夫5 waiting on condition [0x00007f68797d6000]



## 2. 再用jmap 查看heap使用情况

使用jmap -heap pid查看进程堆内存使用情况，包括使用的GC算法、堆配置参数和各代中堆内存使用情况.

jmap -heap 28296

	[running]mqq@10.242.50.116:~$ jmap -heap 28296
	Attaching to process ID 28296, please wait...
	Debugger attached successfully.
	Server compiler detected.
	JVM version is 19.1-b02

	using parallel threads in the new generation.
	using thread-local object allocation.
	Concurrent Mark-Sweep GC

	Heap Configuration:
	   MinHeapFreeRatio = 40
	   MaxHeapFreeRatio = 70
	   MaxHeapSize      = 8589934592 (8192.0MB)
	   NewSize          = 1598029824 (1524.0MB)
	   MaxNewSize       = 1598029824 (1524.0MB)
	   OldSize          = 5439488 (5.1875MB)
	   NewRatio         = 2
	   SurvivorRatio    = 8
	   PermSize         = 21757952 (20.75MB)
	   MaxPermSize      = 85983232 (82.0MB)

	Heap Usage:
	New Generation (Eden + 1 Survivor Space):
	   capacity = 1438253056 (1371.625MB)
	   used     = 931035208 (887.9043655395508MB)
	   free     = 507217848 (483.7206344604492MB)
	   64.73375489215717% used
	Eden Space:
	   capacity = 1278476288 (1219.25MB)
	   used     = 929042232 (886.0037155151367MB)
	   free     = 349434056 (333.2462844848633MB)
	   72.66792827682073% used
	From Space:
	   capacity = 159776768 (152.375MB)
	   used     = 1992976 (1.9006500244140625MB)
	   free     = 157783792 (150.47434997558594MB)
	   1.247350303142945% used
	To Space:
	   capacity = 159776768 (152.375MB)
	   used     = 0 (0.0MB)
	   free     = 159776768 (152.375MB)
	   0.0% used
	concurrent mark-sweep generation:
	   capacity = 6991904768 (6668.0MB)
	   used     = 68837320 (65.64838409423828MB)
	   free     = 6923067448 (6602.351615905762MB)
	   0.9845288556424457% used
	Perm Generation:
	   capacity = 32751616 (31.234375MB)
	   used     = 20263632 (19.324905395507812MB)
	   free     = 12487984 (11.909469604492188MB)
	   61.870632581915956% used

## 3. 使用jstat查询实时内存

如采样时间间隔为250ms，采样数为5。

jstat -gc 28296 250 5

	[running]mqq@10.242.50.116:~$ jstat -gc 28296 250 5
	 S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT   
	156032.0 156032.0  0.0   2812.9 1248512.0 587877.2 6828032.0   70528.6   31984.0 19790.3    131    0.654   2      0.409    1.063
	156032.0 156032.0  0.0   2812.9 1248512.0 592942.4 6828032.0   70528.6   31984.0 19790.3    131    0.654   2      0.409    1.063
	156032.0 156032.0  0.0   2812.9 1248512.0 607178.3 6828032.0   70528.6   31984.0 19790.3    131    0.654   2      0.409    1.063
	156032.0 156032.0  0.0   2812.9 1248512.0 625968.4 6828032.0   70528.6   31984.0 19790.3    131    0.654   2      0.409    1.063
	156032.0 156032.0  0.0   2812.9 1248512.0 650677.8 6828032.0   70528.6   31984.0 19790.3    131    0.654   2      0.409    1.063

或者查看一条：

jstat -gc 28296

## 4. 使用jmap -histo[:live] pid查看堆内存中的对象数目、大小统计直方图

如果带上live则只统计活对象

	jmap -histo:live 28296 | less

> 小知识:
less和more具有相同功能，控空格可查看下一页，但less能支持用上下键进行翻滚


	[running]mqq@10.242.50.116:~$ jmap -histo:live 28296 | less

	 num     #instances         #bytes  class name
	----------------------------------------------
	   1:          4929       17685792  [B
	   2:        391818        6269088  java.lang.Object
	   3:         43527        5223240  java.net.SocksSocketImpl
	   4:         43499        4871888  sun.nio.ch.SocketChannelImpl
	   5:         30224        4296664  <constMethodKlass>
	   6:         30224        3637296  <methodKlass>
	   7:          2413        2774808  <constantPoolKlass>
	   8:         46713        2431056  <symbolKlass>
	   9:         28612        2101328  [C
	  10:         43382        2082336  sun.nio.ch.SocketAdaptor
	  11:          2413        1888880  <instanceKlassKlass>
	  12:         46052        1842080  java.lang.ref.Finalizer
	  13:          2197        1781864  <constantPoolCacheKlass>
	  14:         43638        1396416  java.net.Inet4Address
	  15:         43501        1392032  [Ljava.nio.channels.SelectionKey;
	  16:         43501        1044024  java.net.InetSocketAddress
	  17:          2128        1038928  <methodDataKlass>
	  18:         29082         930624  java.lang.String
	  19:         46056         736896  java.io.FileDescriptor
	  20:         43500         696000  java.nio.channels.spi.AbstractInterruptibleChannel$1
	  21:         43382         694112  sun.nio.ch.OptionAdaptor
	  22:         43382         694112  sun.nio.ch.SocketChannelImpl$1
	  23:         43382         694112  sun.nio.ch.SocketOptsImpl$IP$TCP
	  24:         12670         405440  java.util.concurrent.locks.ReentrantLock$NonfairSync
	  25:         12000         384000  java.util.HashMap$Entry
	  26:          3607         317416  java.lang.reflect.Method
	  27:          2616         272064  java.lang.Class
	  28:          2418         237344  [Ljava.util.HashMap$Entry;
	  29:          3392         230256  [S
	  30:          9392         225408  sun.reflect.generics.tree.SimpleClassTypeSignature
	  31:          3388         216832  com.qq.nami.core.nio.TCPSession
	  32:          4523         212720  [Ljava.lang.Object;
	  33:          5312         212480  java.util.concurrent.ConcurrentHashMap$Segment
	  34:          3704         207840  [[I
	  35:          3264         194216  [I
	  36:          6972         167328  java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject
	  37:          3475         166800  java.util.concurrent.LinkedBlockingQueue
	  38:          9392         156424  [Lsun.reflect.generics.tree.TypeArgument;
	  39:          5312         131832  [Ljava.util.concurrent.ConcurrentHashMap$HashEntry;
	  40:          3175         127000  sun.nio.ch.SelectionKeyImpl


class name是对象类型，说明如下：

	B  byte
	C  char
	D  double
	F  float
	I  int
	J  long
	Z  boolean
	[  数组，如[I表示int[]
	[L+类名 其他对象

## 5. 最后，如果还是看不出问题，可以调大heap再观察一段时间

一般来说，问题可能是内存泄漏，也可能是因为服务对象太多，导致空间不足，引发频繁地gc。