---
layout: post
title: Java 内存调试工具
category: java
comments: false
---

## 1. java 内置工具汇总
JDK本身提供了很多方便的JVM性能调优监控工具，如下：

- jconsole
- jvisualvm
- jps
- jstack
- jmap
- jhat
- jstat
- jinfo
	观察进程运行环境参数，包括Java System属性和JVM命令行参数.(系统崩溃了？jinfo可以从core文件里面知道崩溃的Java应用程序的配置信息。)

### 1.1 jps (proccess status tool)
jps主要用来输出JVM中运行的进程状态信息。

	jps [options] [hostid]

	如果不指定hostid就默认为当前主机或服务器。

参数：

	-q 不输出类名、Jar名和传入main方法的参数
	-m 输出传入main方法的参数
	-l 输出main类或Jar的全限名
	-v 输出传入JVM的参数

	root@ubuntu:/# jps -m -l
	2458 org.artifactory.standalone.main.Main /usr/local/artifactory-2.2.5/etc/jetty.xml
	29920 com.sun.tools.hat.Main -port 9998 /tmp/dump.dat
	3149 org.apache.catalina.startup.Bootstrap start
	30972 sun.tools.jps.Jps -m -l
	8247 org.apache.catalina.startup.Bootstrap start
	25687 com.sun.tools.hat.Main -port 9999 dump.dat
	21711 mrf-center.jar

### 1.2 jstack
 jstack可以定位到线程堆栈，根据堆栈信息我们可以定位到具体代码，所以它在JVM性能调优中使用得非常多。

### 1.3 jmap （Memory Map）
jmap用来查看堆内存使用状况，一般结合jhat使用。

jmap语法格式如下：

	jmap [option] pid
	jmap [option] executable core
	jmap [option] [server-id@]remote-hostname-or-ip
    如果运行在64位JVM上，可能需要指定-J-d64命令选项参数。

例子：

 	jmap  pid

    打印进程的类加载器和类加载器加载的持久代对象信息，输出：类加载器名称、对象是否存活（不可靠）、对象地址、父类加载器、已加载的类大小等信息

	[root@qd251 bin]# jmap 1802
	Attaching to process ID 1802, please wait...
	Debugger attached successfully.
	Server compiler detected.
	JVM version is 25.73-b02
	0x0000000000400000      7K      /app/src/jdk1.8.0_73/bin/java
	0x000000356cc00000      141K    /lib64/ld-2.5.so
	0x000000356d000000      1685K   /lib64/libc-2.5.so
	0x000000356d400000      22K     /lib64/libdl-2.5.so
	0x000000356d800000      146K    /lib64/libpthread-2.5.so
	0x000000356dc00000      600K    /lib64/libm-2.5.so
	0x000000356e400000      52K     /lib64/librt-2.5.so
	0x00002b606d64c000      100K    /app/src/jdk1.8.0_73/lib/amd64/jli/libjli.so
	0x00002b606d865000      16521K  /app/src/jdk1.8.0_73/jre/lib/amd64/server/libjvm.so
	0x00002b606e946000      64K     /app/src/jdk1.8.0_73/jre/lib/amd64/libverify.so
	0x00002b606eb55000      220K    /app/src/jdk1.8.0_73/jre/lib/amd64/libjava.so
	0x00002b606edc0000      121K    /app/src/jdk1.8.0_73/jre/lib/amd64/libzip.so
	0x00002b608a855000      48K     /app/src/jdk1.8.0_73/jre/lib/amd64/libmanagement.so
	0x00002b608aa5e000      113K    /app/src/jdk1.8.0_73/jre/lib/amd64/libnet.so
	0x00002b608ac75000      90K     /app/src/jdk1.8.0_73/jre/lib/amd64/libnio.so

使用jmap -heap pid查看进程堆内存使用情况，包括使用的GC算法、堆配置参数和各代中堆内存使用情况。

	[root@qd251 bin]# jmap -heap 1802	                                                  
	Attaching to process ID 1802, please wait...
	Debugger attached successfully.
	Server compiler detected.
	JVM version is 25.73-b02

	using thread-local object allocation.
	Parallel GC with 8 thread(s)

	Heap Configuration:
	   MinHeapFreeRatio         = 0
	   MaxHeapFreeRatio         = 100
	   MaxHeapSize              = 4198498304 (4004.0MB)
	   NewSize                  = 88080384 (84.0MB)
	   MaxNewSize               = 1399324672 (1334.5MB)
	   OldSize                  = 176160768 (168.0MB)
	   NewRatio                 = 2
	   SurvivorRatio            = 8
	   MetaspaceSize            = 21807104 (20.796875MB)
	   CompressedClassSpaceSize = 1073741824 (1024.0MB)
	   MaxMetaspaceSize         = 17592186044415 MB
	   G1HeapRegionSize         = 0 (0.0MB)

	Heap Usage:
	PS Young Generation
	Eden Space:
	   capacity = 31981568 (30.5MB)
	   used     = 27342048 (26.075408935546875MB)
	   free     = 4639520 (4.424591064453125MB)
	   85.49314405097336% used
	From Space:
	   capacity = 524288 (0.5MB)
	   used     = 131072 (0.125MB)
	   free     = 393216 (0.375MB)
	   25.0% used
	To Space:
	   capacity = 524288 (0.5MB)
	   used     = 0 (0.0MB)
	   free     = 524288 (0.5MB)
	   0.0% used
	PS Old Generation
	   capacity = 176160768 (168.0MB)
	   used     = 45306160 (43.20732116699219MB)
	   free     = 130854608 (124.79267883300781MB)
	   25.718643551781064% used

还有一个很常用的情况是：用jmap把进程内存使用情况dump到文件中，再用jhat分析查看。jmap进行dump命令格式如下：

	jmap -dump:format=b,file=dumpFileName
   
我一样地对上面进程ID为21711进行Dump：

	root@ubuntu:/# jmap -dump:format=b,file=/tmp/dump.dat 21711     
	Dumping heap to /tmp/dump.dat ...
	Heap dump file created

### 1.4 jhat（Java Heap Analysis Tool）
（接上节）dump出来的文件可以用MAT、VisualVM等工具查看，这里用jhat查看：

	root@ubuntu:/# jhat -port 9998 /tmp/dump.dat
	Reading from /tmp/dump.dat...
	Dump file created Tue Jan 28 17:46:14 CST 2014
	Snapshot read, resolving...
	Resolving 132207 objects...
	Chasing references, expect 26 dots..........................
	Eliminating duplicate references..........................
	Snapshot resolved.
	Started HTTP server on port 9998
	Server is ready.
 
 然后就可以在浏览器中输入主机地址:9998查看了。

### 1.5 jstat（JVM统计监测工具）

语法格式如下：

	jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]
    
vmid是虚拟机ID，在Linux/Unix系统上一般就是进程ID。interval是采样时间间隔。count是采样数目。比如下面输出的是GC信息，采样时间间隔为250ms，采样数为4：

	root@ubuntu:/# jstat -gc 21711 250 4
	 S0C    S1C    S0U    S1U      EC       EU        OC         OU       PC     PU    YGC     YGCT    FGC    FGCT     GCT   
	192.0  192.0   64.0   0.0    6144.0   1854.9   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
	192.0  192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
	192.0  192.0   64.0   0.0    6144.0   1972.2   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649
	192.0  192.0   64.0   0.0    6144.0   2109.7   32000.0     4111.6   55296.0 25472.7    702    0.431   3      0.218    0.649   

各列含义：

	S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
	EC、EU：Eden区容量和使用量
	OC、OU：年老代容量和使用量
	PC、PU：永久代容量和使用量
	YGC、YGT：年轻代GC次数和GC耗时
	FGC、FGCT：Full GC次数和Full GC耗时
	GCT：GC总耗时

## 2. 实例

使用jstack查看进程执行栈。

	[root@qd251 ~]# jstack 24270
	2016-11-03 15:55:20
	Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.73-b02 mixed mode):

	"Attach Listener" #33 daemon prio=9 os_prio=0 tid=0x0000000012837000 nid=0x1c73 waiting on condition [0x0000000000000000]
	   java.lang.Thread.State: RUNNABLE

	"OkHttp ConnectionPool" #32 daemon prio=5 os_prio=0 tid=0x000000000a046000 nid=0x5efa in Object.wait() [0x00002b28e243b000]
	   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	        at java.lang.Object.wait(Native Method)
	        at java.lang.Object.wait(Object.java:460)
	        at com.squareup.okhttp.ConnectionPool.performCleanup(ConnectionPool.java:305)
	        - locked <0x00000006c5c73600> (a com.squareup.okhttp.ConnectionPool)
	        at com.squareup.okhttp.ConnectionPool.runCleanupUntilPoolIsEmpty(ConnectionPool.java:242)
	        at com.squareup.okhttp.ConnectionPool.access$000(ConnectionPool.java:54)
	        at com.squareup.okhttp.ConnectionPool$1.run(ConnectionPool.java:97)
	        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	        at java.lang.Thread.run(Thread.java:745)

	"Okio Watchdog" #31 daemon prio=5 os_prio=0 tid=0x000000000a040800 nid=0x5ef9 in Object.wait() [0x00002b28e253c000]
	   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	        at java.lang.Object.wait(Native Method)
	        at java.lang.Object.wait(Object.java:460)
	        at okio.AsyncTimeout.awaitTimeout(AsyncTimeout.java:309)
	        - locked <0x00000006c5c8bc88> (a java.lang.Class for okio.AsyncTimeout)
	        at okio.AsyncTimeout.access$000(AsyncTimeout.java:40)
	        at okio.AsyncTimeout$Watchdog.run(AsyncTimeout.java:272)

	"commons-pool-EvictionTimer" #29 daemon prio=5 os_prio=0 tid=0x0000000009d81800 nid=0x5ef7 in Object.wait() [0x00002b28e2239000]
	   java.lang.Thread.State: TIMED_WAITING (on object monitor)
	        at java.lang.Object.wait(Native Method)
	        at java.util.TimerThread.mainLoop(Timer.java:552)
	        - locked <0x00000006c5c7c620> (a java.util.TaskQueue)
	        at java.util.TimerThread.run(Timer.java:505)

	"Thread-[mmfreebook]" #27 prio=5 os_prio=0 tid=0x00002b28e42ae800 nid=0x5ef5 waiting on condition [0x00002b28e2f4d000]
	   java.lang.Thread.State: WAITING (parking)
	        at sun.misc.Unsafe.park(Native Method)
	        - parking to wait for  <0x00000006c5c86ea0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1088)
	        at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	        at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	        at java.lang.Thread.run(Thread.java:745)

## 参考
>[JVM性能调优监控工具jps、jstack、jmap、jhat、jstat使用详解](http://www.open-open.com/lib/view/open1390916852007.html)