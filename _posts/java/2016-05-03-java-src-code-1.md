---
layout: post
title: Java源码阅读1
category: java
comments: false
---

## 阅读Java源码的初衷

1. 装b
2. 满足自己的好奇心
3. 提升自己的编码能力
4. 为下次面试做好准备

## 执行过程

阅读代码需要有编程基础和足够的耐心，最后要有记录和思考，虽说如此，但我任务这个比看ACM和IEEE会议上的paper要容易一些吧。

1. 选好IDE和需要阅读的源代码： Intellij idea (或eclipse) + JDK 1.7
1. 先看一些大牛门的建议，少走一些弯路。
  > [Java源码阅读的真实体会](http://zwchen.iteye.com/blog/1154193)  
  [Java 推荐读物与源代码阅读](http://www.360doc.com/content/10/0116/16/724027_13720736.shtml)  
  [如何更有效地学习开源项目的代码？](https://www.zhihu.com/question/19637879)

1. 观看顺序建议
    - 先熟悉代码结构，有一个大局观和查看计划；
    - 再看类库预热，如：String类的实现（core包），Util中基本数据结构的实现（util包），并发(concurrency包)，有关文本的（text包），有关网络的http(http包)等等;
    - 然后再看JVM部分代码，这里需要先理解JVM的工作原理，尽量读懂代码，但也不要过分深入，除非工作内容和这个密切相关；
    - 最后，一定要整理回顾，将所看的内容整理成文章，用脑记下来。

1. 有时间去看看spring的源码，据说很赞。


## Java代码阅读之路
第一天：java.util

- 这里会和一个叫java集合框架（Java Collections Framework)的东西打交道,站内已有文章介绍过了。
- Iterator设计的初衷是用于替换Enumeration的，与Enum的不同之处：1、允许删除元素；2、提高了函数的命名规范。注意，Iterator的接口只有三个函数：hasNext()/next()/remove()。
- replete “充满”的意思 （我去，还要在旁边准备个字典，真丢母校的脸）
- AbstractCollection.java: 在关于迭代器到数组之间的转换时，会频繁用到一个函数：Arrays.copyOf(r, i); 可以研究一下Arrays。

### java/util/Arrays.java

- 注意到在JDK库里面，不管函数是否为static，函数名都是小写开头。
- pivot n 中心点/枢轴; adj 枢轴的; vt&vi 以…为中心旋转
- sort()的排序算法：Dual-Pivot Quicksort algorithm
- 快排算法核心(one-pivot Quicksort implementations）：1、先选一个点，当作pivot；2、用快排算法将这个点放在正确的位置上；3、对该点前面的数组和后面的数组递归调用函数。
- [双轴快排算法](http://stackoverflow.com/questions/20917617/whats-the-difference-of-dual-pivot-quick-sort-and-quick-sort):  
1、对于小数组（<47 变了),直接用插入排序；  
2、选择两个中心点P1和P2，保证P1小于P2；  
3、分割剩余的数组：第一部分——小于P1（begin至L-1）；第二部分——不小于P1并小于P2（L至K-1）；第三部分——大于P2（G至end）；第四部分——剩余的那一堆（K至G-1）；  

      /*  
       * Partitioning:(充满了geek精神的一段注释)  
       *  
       *   left part           center part                   right part  
       * +--------------------------------------------------------------+  
       * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |  
       * +--------------------------------------------------------------+  
       *               ^                          ^       ^  
       *               |                          |       |  
       *              less(L)                     k     great(G)  
       *  
       * Invariants:  
       *  
       *              all in (left, less)   < pivot1  
       *    pivot1 <= all in [less, k)     <= pivot2  
       *              all in (great, right) > pivot2  
       *  
       * Pointer k is the first index of ?-part.  
       */  
4、算法不断将第四部分的数据往其他部分放，L、K和G也往相对应的方向增长；  
5、重复4直到K<=G;  
6、P1放在L的位置，P2放在G（或K）的位置；  
7、对第一到第三部分的数据都递归调用该函数。

- Dual-Pivot Quicksort algorithm： This algorithm offers O(n log(n)) performance on many data sets that cause other quicksorts to degrade to quadratic performance, and is typically faster than traditional (one-pivot) Quicksort implementations.
