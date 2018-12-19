---
layout: post
title: OOM vs StackOverflow
category: java
comments: true
---

## 一、stackoverflow

每当java程序启动一个新的线程时，java虚拟机会为他分配一个栈，java栈以帧为单位保持线程运行状态；当线程调用一个方法是，jvm压入一个新的栈帧到这个线程的栈中，只要这个方法还没返回，这个栈帧就存在。 

如果方法的嵌套调用层次太多(如递归调用), 随着java栈中的帧的增多，最终导致这个线程的栈中的所有栈帧的大小的总和大于-Xss设置的值，而产生生StackOverflowError溢出异常。

复现：函数递归调用自己。启动时把-Xss(设置每个线程的堆栈大小)设置得小一点。

注意：如果是新建线程，报出的错会是OOM！

注意：如果你把虚拟机参数Xss调大了,每个线程的占用的栈空间也就变大了，那么可以建立的线程数量必然减少，有可能诱发OOM。（当前大部分的Java 虚拟机都可动态扩展）

## 二、outofmemory

这种错误发生的情况就比较多。（程序计数器内存区域是唯一一个在Java 虚拟机规范中没有规定任何OutOfMemoryError 情况的区域。）

### 2.1 栈内存溢出

java程序启动一个新线程时，没有足够的空间为线程分配java栈，JVM则抛出OutOfMemoryError异常。

### 2.2 堆内存溢出

java堆用于存放对象的实例，当需要为对象的实例分配内存时，而堆的占用已经达到了设置的最大值(通过-Xmx 设置JVM最大可用内存)设置最大值，则抛出OutOfMemoryError异常。

### 2.3 方法区内存溢出

方法区（也是所有线程共享）用于存放java类的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。在类加载器加载class文件到内存中的时候，JVM会提取其中的类信息，并将这些类信息放到方法区中。 

当需要存储这些类信息，而方法区的内存占用又已经达到最大值（通过-XX:MaxPermSize）；将会抛出OutOfMemoryError异常对于这种情况的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出。

可以借助CGLib直接操作字节码运行，生成大量的动态类。


## 三、例子

### 3.1 栈溢出

例如，通过递归调用方法,不停的产生栈帧,一直把栈空间堆满,直到抛出异常：

    package com.demo3;

    public class OOMTest {
        public void stackOverFlowMethod() {
            stackOverFlowMethod();
        }     

        public static void main(String... args) {
            OOMTest oom = new OOMTest();
            oom.stackOverFlowMethod();
        }
    }

结果：

    Exception in thread "main" java.lang.StackOverflowError
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    at com.demo3.OOMTest.stackOverFlowMethod(OOMTest.java:5)
    .....

### 3.2 内存溢出示例

内存溢出是指当我们新建一个实例对象时，实例对象所需占用的内存空间大于堆的可用空间。
如果出现了内存溢出问题，这往往是程序本生需要的内存大于了我们给虚拟机配置的内存，这种情况下，我们可以采用调大-Xmx来解决这种问题。

    package com.demo3;

    import java.util.ArrayList;
    import java.util.List;

    public class OOMTest {

        public static void main(String[] args) {
            List<byte[]> buffer = new ArrayList<byte[]>();
            buffer.add(new byte[10 * 1024 * 1024]);
        }

    }

通过如下命令运行程序：
    
    java -verbose:gc -Xmn10M -Xms20M -Xmx20M -XX:+PrintGC OOMTest

Xmn设置年轻代大小为10M, Xms设置初始内存为20M，Xmx整个JVM内存上限20M(整个JVM内存大小=年轻代大小 + 年老代大小 + 持久代大小)

输出结果：

    [GC 836K->568K(19456K), 0.0234380 secs]
    [GC 568K->536K(19456K), 0.0009309 secs]
    [Full GC 536K->463K(19456K), 0.0085383 secs]
    [GC 463K->463K(19456K), 0.0003160 secs]
    [Full GC 463K->452K(19456K), 0.0062013 secs]
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
        at com.demo3.OOMTest.main(OOMTest.java:10)

## 四、联系和区别

如果在一个线程计算过程中不允许有更大的本地方法栈，那么JVM就抛出StackOverflowError。

如果本地方法栈可以动态地扩展，并且本地方法栈尝试过扩展了，但是没有足够的内容分配给它，再或者没有足够的内存为线程建立初始化本地方法栈，那么JVM抛出的就是OutOfMemoryError。

## REF
> [https://blog.csdn.net/weixin_40667145/article/details/78556182](https://blog.csdn.net/weixin_40667145/article/details/78556182)