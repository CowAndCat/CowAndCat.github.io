---
layout: post
title: Java 面试题4
category: java
comments: false
---
###Java面试题
>http://www.nowcoder.com/ta/review-java


1. 什么是Java虚拟机？为什么Java被称作是“平台无关的编程语言”？
2. JDK和JRE的区别是什么？
3. "static"关键字是什么意思？Java中是否可以覆盖(override)一个private或者是static的方法？
4. Java支持的数据类型有哪些？什么是自动拆装箱？
5. Java中的方法覆盖(Overriding)和方法重载(Overloading)是什么意思？
6. Java支持多继承么？如果不支持，如何实现。（多继承有两种方式 , 一是接口，二是内部类.）
7. 什么是值传递和引用传递？（**值传递**就是在方法调用的时候，实参是将自己的一份拷贝赋给形参，在方法内，对该参数值的修改不影响原来实参。**引用传递**是在方法调用的时候，实参将自己的地址传递给形参，此时方法内对该参数值的改变，就是对该实参的实际操作。）
8. 进程和线程的区别是什么？
	1. 线程是进程的一个组成部分 .
	2. 进程的多个线程都在进程地址空间活动 .
	3. 系统资源是分配给进程的，线程需要资源时，系统从进程的资源里分配给线程 .
	4. 处理机调度的基本单位是线程 .
9. 创建线程有几种不同的方式？哪种更好？为什么?
10. 概括的解释下线程的几种可用状态
11. HashMap和Hashtable有什么区别？
	1. HashTable 基于 Dictionary 类，而 HashMap 是基于 AbstractMap 。 Dictionary 是任何可将键映射到相应值的类的抽象父类，而 AbstractMap 是基于 Map 接口的实现，它以最大限度地减少实现此接口所需的工作。
	2. HashMap 的 key 和 value 都允许为 null ，而 Hashtable 的 key 和 value 都不允许为 null 。 HashMap 遇到 key 为 null 的时候，调用 putForNullKey 方法进行处理，而对 value 没有处理； Hashtable 遇到 null ，直接返回 NullPointerException 。
	3. Hashtable 是同步的，而 HashMap 是非同步的，但是我们也可以通过 Collections.synchronizedMap(hashMap), 使其实现同步。
12. java中的HashMap的工作原理是什么？  
    HashMap 内部是通过一个数组实现的，只是这个数组比较特殊，数组里存储的元素是一个 Entry 实体 (jdk 8 为 Node) ，这个 Entry 实体主要包含 key 、 value 以及一个指向自身的 next 指针。 HashMap 是基于 hashing 实现的，当我们进行 put 操作时，根据传递的 key 值得到它的 hashcode ，然后再用这个 hashcode 与数组的长度进行模运算，得到一个 int 值，就是 Entry 要存储在数组的位置（下标）；当通过 get 方法获取指定 key 的值时，会根据这个 key 算出它的 hash 值（数组下标），根据这个 hash 值获取数组下标对应的 Entry ，然后判断 Entry 里的 key ， hash 值或者通过 equals() 比较是否与要查找的相同，如果相同，返回 value ，否则的话，遍历该链表（有可能就只有一个 Entry ，此时直接返回 null ），直到找到为止，否则返回 null 。  
	HashMap 之所以在每个数组元素存储的是一个链表，是为了解决 hash 冲突问题，当两个对象的 hash 值相等时，那么一个位置肯定是放不下两个值的，于是 hashmap 采用链表来解决这种冲突， hash 值相等的两个元素会形成一个链表。
13. java中的四种引用
    - 强引用:垃圾回收器绝不会回收它。当内存空 间不足，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题.
    - 软引用(SoftReference):如果内存空间足够，垃圾回收器就不会回收它，如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。  
　　软引用可以和一个引用队列(ReferenceQueue)联合使用，如果软引用所引用的对象被垃圾回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。
	- 弱引用(WeakReference)  
	弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，**都会回收**它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
	- 虚引用(PhantomReference)
　　"虚引用"顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。  
   虚引用主要用来跟踪对象被垃圾回收的活动。  
虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列(ReferenceQueue)联合使用。当垃 圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是 否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

14. JVM内存分区，每个区的作用是什么?  
![1](/images/201511/sdkmem.PNG "sdk内存区")
![2](/images/201511/jvm.png "jvm内存区")
	- 程序计数器:当前线程所执行字节码的行号指示器，所以它是私有的.
	- Java栈（Java虚拟机栈）:Java栈与线程的生命周期相同，java栈中存放的是一个个栈帧。栈帧中存放的是局部变量表、操作数栈、指向运行时常量池的引用、方法返回值地址和附加信息。当jvm创建一个线程时，Java栈也随之创建（因此它也是线程私有），线程执行一个方法时就会创建一个栈与之对应的帧并压入栈中，方法执行结束，栈帧出栈。
	- 本地方法栈:为Native方法服务。
	- java堆：虚拟机启动时创建，线程共享，用于存储数组以及对象。（-Xmx和-Xms控制）
	- 方法区（非堆）：存储常量、静态变量、已经被虚拟机加载的类信息（包括类的名称、方法信息、字段信息）等。
15. java垃圾收集机制？收集方法?（分代收集：复制+标记-清除-整理）
16. Java接口与抽象类的区别，能否在接口中声明final方法、为什么
17. java类加载过程？解释双亲委派模型。  
	java 类加载需要经历一下 7 个过程：   
	1- 加载   
	加载是类加载的第一个过程，在这个阶段，将完成一下三件事情：
	1. 通过一个类的权限定名获取该类的二进制流。
	2. 将该二进制流中的静态存储结构转化为方法去运行时数据结构。 3. 在内存中生成该类的 Class 对象，作为该类的数据访问入口。

	2- 验证   
	验证的目的是为了确保 Class 文件的字节流中的信息不回危害到虚拟机 . 在该阶段主要完成以下四种验证 :
	1. 文件格式验证：验证字节流是否符合 Class 文件的规范，如主次版本号是否在当前虚拟机范围内，常量池中的常量是否有不被支持的类型 .
	2. 元数据验证 : 对字节码描述的信息进行语义分析，如这个类是否有父类，是否集成了不被继承的类等。
	3. 字节码验证：是整个验证过程中最复杂的一个阶段，通过验证数据流和控制流的分析，确定程序语义是否正确，主要针对方法体的验证。如：方法中的类型转换是否正确，跳转指令是否正确等。
	4. 符号引用验证：这个动作在后面的解析过程中发生，主要是为了确保解析动作能正确执行。

	3- 准备  
	准备阶段是**为类的静态变量分配内存并将其初始化为默认值**，这些内存都将在方法区中进行分配。准备阶段不分配类中的实例变量的内存，实例变量将会在对象实例化时随着对象一起分配在 Java 堆中。
	    public static int value=123;//在准备阶段value初始值为0 。在初始化阶段才会变为123 。

	4- 解析
	该阶段主要完成符号引用到直接引用的转换动作。解析动作并不一定在初始化动作完成之前，也有可能在初始化之后。

	5- 初始化
	初始化是类加载的最后一步，前面的类加载过程，除了在加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导和控制。到了初始化阶段，才真正开始执行类中定义的 Java 程序代码。

	**双亲委派模型**：当一个类收到了类加载请求时，不会自己先去加载这个类，而是将其委派给父类，由父类去加载，如果此时父类不能加载，反馈给子类，由子类去完成类的加载。
18. java内存模型
19. 什么是反射，反射机制
20. java泛型、泛型与类型擦除
21. java中的四大特性
22. volatile 变量和 atomic 变量有什么不同？  
	在Java 中除了 long 和 double 之外的所有基本类型的读和赋值，都是原子性操作。而64位的long 和 double 变量由于会被JVM当作两个分离的32位来进行操作，所以不具有原子性，会产生字撕裂问题。但是当你定义long或double变量时，如果使用 volatile关键字，就会获到（简单的赋值与返回操作的）原子性（注意，在Java SE5之前，volatile一直不能正确的工作）。见第四版《Thinking in java》第21章并发。

	volatile关键字确保了应用中的可视性。如果你将一个域声明为volatile，那么只要这个域产生了写操作，那么所有的读操作就都可以看到这个修改。

	原子操作就是不能被线程调度机制中断的操作
23. ThrealLocal是什么，原理
24. sleep() 和 wait () 方法有什么区别？  
wait后进入等待锁定池，使用此对象发出的notify 或 notifyAll 方法获得对象锁进行就绪状态（非运行状态）。

	sleep和wait的主要区别是是否释放锁、监视对象，sleep调用时会继续保持（其它线程阻塞等待）；wait调用时会释放（另一个线程可获得锁继续执行）。此外，wait用于多线程间通信，sleep 用于暂停特定时间执行。
25. java JNI相关
26. 线程同步的方法有哪些
27. java socket编程
28. Callable和Runnable的区别
29. ConcurrentHashMap原理
	ConcurrenHashMap 可以说是 HashMap 的升级版， ConcurrentHashMap 是线程安全的， ConcurrentHashMap 是采用分离锁的方式，它并没有对整个 hash 表进行锁定，而是局部锁定，也就是说当一个线程占有这个局部锁时，不影响其他线程对 hash 表其他地方的访问。
30. 线程池的概念、好处、常见的线程池举例
31. 原子性与可见性
32. 如何判断一个对象是否存活
33. java IO，NIO
34. 接口和抽象类的区别是什么 ?
	1. 接口中所有的方法隐含的都是抽象的。而抽象类则可以同时包含抽象和非抽象的方法。
	2. 类可以实现很多个接口，但是只能继承一个抽象类。
	3. 类如果要实现一个接口，它必须要实现接口声明的所有方法。但是，类可以不实现抽象类声明的所有方法，当然，在这种情况下，类也必须得声明成是抽象的。
	4. 抽象类可以在不提供接口方法实现的情况下实现接口。
	5. Java 接口中声明的变量默认都是 final 的。抽象类可以包含非 final 的变量。
	6. Java 接口中的成员函数默认是 public 的。抽象类的成员函数可以是 private ， protected 或者是 public 。
	7. 接口是绝对抽象的，不可以被实例化 (java 8 已支持在接口中实现默认的方法 ) 。抽象类也不可以被实例化，但是，如果它包含 main 方法的话是可以被调用的。
35. 什么是类加载器，类加载器有哪些 ?  
	实现通过类的权限定名获取该类的二进制字节流的代码块叫做类加载器。
	主要有一下四种类加载器 :
	1. 启动类加载器 (Bootstrap ClassLoader) 用来加载 java 核心类库，无法被 java 程序直接引用。
	2. 扩展类加载器 (extensions class loader): 它用来加载 Java 的扩展库。 Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。
	3. 系统类加载器（ system class loader ）：它根据 Java 应用的类路径（ CLASSPATH ）来加载 Java 类。一般来说， Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader() 来获取它。
	4. 用户自定义类加载器，通过继承 java.lang.ClassLoader 类的方式实现。
36. 简述 java 内存分配与回收策率以及 Minor GC 和 Major GC。  
	1. 对象优先在堆的 Eden 区分配。
	2.    大对象直接进入老年代 .
	3.    长期存活的对象将直接进入老年代 .  
	当 Eden 区没有足够的空间进行分配时，虚拟机会执行一次 Minor GC.Minor Gc 通常发生在新生代的 Eden 区，在这个区的对象生存期短，往往发生 Gc 的频率较高，回收速度比较快 ;Full Gc/Major GC 发生在老年代，一般情况下，触发老年代 GC 的时候不会触发 Minor GC, 但是通过配置，可以在 Full GC 之前进行一次 Minor GC 这样可以加快老年代的回收速度。
37. fail-fast 与 fail-safe 有什么区别？  
	Iterator 的 fail-fast 属性与当前的集合共同起作用，因此它不会受到集合中任何改动的影响。 Java.util 包中的所有集合类都被设计为 fail->fast 的，而 java.util.concurrent 中的集合类都为 fail-safe 的。当检测到正在遍历的集合的结构被改变时， Fail-fast 迭代器抛出 ConcurrentModificationException ，而 fail-safe 迭代器从不抛出 ConcurrentModificationException 。
38. Array 和 ArrayList 有何区别？什么时候更适合用 Array ？
	1.    Array 可以容纳基本类型和对象，而 ArrayList 只能容纳对象。
	2.    Array 是指定大小的，而 ArrayList 大小是固定的

39. Thread 类中的 start() 和 run() 方法有什么区别？  
	start() 方法被用来启动新创建的线程，而且 start() 内部调用了 run() 方法，这和直接调用 run() 方法的效果不一样。当你调用 run() 方法的时候，只会是在原来的线程中调用，没有新的线程启动， start() 方法才会启动新线程。
40. String的intern()函数  
	`public native String intern();`  
	Returns a canonical(规范的) representation for the string object.

	A pool of strings, initially empty, is maintained privately by the class String.

	When the intern method is invoked, if the pool already contains a string equal to this String object as determined by the equals(Object) method(判断调用者是否在pool中已存在）, then the string from the pool is returned（存在就返回pool中的the string）. Otherwise, this String object is added to the pool and a reference to this String object is returned(否则就将对象添加到pool并返回引用）.

    It follows that for any two strings s and t, s.intern() == t.intern() is true if and only if s.equals(t) is true.

###数据结构与算法
1. 九个排序算法，时间复杂度，什么情况下用哪种排序。
2. 链表
3. 栈
4. 队列
5. 二叉树，遍历方式的实现，递归与非递归版
6. 图：BFS，DFS，最短路径等
7. 字符串匹配，kmp算法
8. 二分查找，hash表
>理解数据结构原理后，多做题，剑指offer，程序员面试宝典等
###计算机网络
1. tcp三次握手，四次挥手（常问）
2. tcp可靠原理，流量控制，拥塞控制
3. tcp，udp原理
4. OSI分层与TCP/P分层,每层作用
5. 解释ARP,ICMP
5. DNS域名解析
6. 交换机，网关，路由器概念，作用
7. TCP连接管理，优化
8. Http请求头，请求报文，相应报文，状态码及含义
9. IP地址的分类，无分类CIDR，划分子网，ip数据报格式，ip网络地址及广播地址的计算（笔试常考）
10. 说一下在浏览器输入www.xxx.com背后的原理(dns,http,tcp相关知识)
11. URI与URL
12. web缓存，代理，https等（了解）
13. Http怎么处理长连接，http有无状态，如何保持状态
14. Cookie和Session(知道最好)
>参考书籍《计算机网络》、《HTTP权威指南》
###操作系统
1. 死锁的必要条件，怎么处理死锁。
2. 进程的几种状态
3. IPC几种通信方式。
4. 什么是虚拟内存。
5. 虚拟地址、逻辑地址、线性地址、物理地址的区别
6. 内存管理方式
7. 进程调度的一些算法策略
8. 了解linux吗，linux常用命令，内核原理
>《深入理解操作系统》
