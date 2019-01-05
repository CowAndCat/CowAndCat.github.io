---
layout: post
title: Java ConcurrentHashMap
category: java
comments: false
---

本文将介绍
1. HashMap,Hashtable和ConcurrentHashMap的异同
2. ConcurrentHashMap 在JDK1.7和1.8里实现异同
3. 注意事项
4. 并发读的实现。

# 一、HashMap,Hashtable和ConcurrentHashMap的异同

## 1.1 HashMap

- HashMap 默认不是线程安全的。（线程安全是指，在对象被多个线程访问时仍然有效，不管运行时环境如何排列，线程都不需要任何额外的同步。）
- HashMap允许null key 和value，而Hashtable不允许。
- 因为线程安全的问题，HashMap效率比HashTable的要高。应用程序一般在更高的层面上实现了保护机制，而不是依赖于这些底层数据结构的同步。


## 1.2 Hashtable
1. Hashtable是线程安全的，内部方法基本都是synchronized.
2. 不允许null值存在
尽管Hashtable和HashMap的结构非常类似，但是他们之间并没有多大联系。

## 1.3 ConcurrentMap
1. 当多线程对ConcurrentHashMap进行操作时，不是完全锁住整个map,而是锁住相应的segment（锁分离）.这会提高并发效率。（1.8的实现方式换成了Node数组，没有segment）
2. 不允许null键存在。
3. 缺点：当遍历ConcurrentMap中的元素，需要获取所有的segment的锁，使用遍历时慢。

## 1.4 其他
最终可用的线程安全版本Map实现是ConcurrentHashMap/ConcurrentSkipListMap/Hashtable/Properties四个，但是Hashtable是过时的类库，因此如果可以的应该尽可能的使用ConcurrentHashMap和ConcurrentSkipListMap。

当length=2^n时，index=hashcode & (length-1) == hashcode % length。由于与(&)是比取模(%)更高效的操作，因此Java中采用hash值与（数组大小length-1)后取与来确定数组索引的。

如果长度不为2的幂次方，会造成索引的浪费，运算也会变复杂。

# 二、ConcurrentHashMap

### 2.1 Design goal (java8)
The primary design goal of this hash table is to maintain concurrent readability (typically method get(), but also iterators and related methods) while minimizing update contention. Secondary goals are to keep space consumption about the same or better than java.util.HashMap, and to support high initial insertion rates on an empty table by many threads.
(主要目标是保持并发可读，同时最小化更新的竞争；其次是用不多于hashMap的空间，增加初始化的insert操作)

### 2.2 Data structures
每一个key-value对存储在一个Node，看起来就像一个bucketed hash table.

子结构：

- TreeNode: 平衡树结构
- TreeBin: 一堆TreeNodes(不用TreeMaps的主要原因，是因为可能存在没有实现compareTo的泛型T，而它可以使用hashCode来排序)(hashcode=-1)
- ForwardingNode: 当resizing时候，放在bins的头上(-2)
- ReservationNode: 当构造值时的占位符（-3）

后面三个都不会存数据，hash值始终为负值。

### 2.3 操作

每次在空bin上的首个Node的插入操作都使用CAS（Compare-and-swap）来执行，其他的更新操作（insert, delete and replace）都要求locks. 这里有个巧妙的地方：不会将锁加到整个bin上，而是加到每个bin的第一个node上。这样可以节省临界区域。但是这个巧妙还需要做一层校验：当一个node被锁住，任何的update操作都要先校验这个node是不是首个node，如果不是就不断重试。因为新node都是加到list后面，除非是被删或被扩容。

### 2.4 Traversal scheme
Any thread noticing an overfull bin may assist in resizing after the initiating thread allocates and sets up the replacement array.（线程会帮着resizing超容的bin）

Resizing proceeds by transferring bins, one by one, from the table to the next table.


因为使用2的幂次方进行扩容，element要么待在原来的bin，要么就移动2的幂次方个位置。平均来说，当一个table扩容一倍，只有六分之一的节点需要clone. 迁移结束后，原来bin只包含一个键值为nextTabel的forwardingNode。

TableStack：因为移动Node的时候会打乱顺序，借助这个结构能保证不遗漏Node.

TreeBin: 当bin里元素个数超过8，将list转tree;当个数小于6，tree转list.

### 2.5 sizeCtl变量
- 保证resizing不会重叠
- -1 表示初始化
- -1-num 表示resize，num表示the number of active resizing thread.
- 小于0 表示下次扩容的大小


## 三、不同JDK的实现异同

### 3.1 JDK6和JDK7的实现

#### 3.1.1 设计思路

在这两个版本中，ConcurrentHashMap采用了分段锁的设计，只有在同一个分段内才存在竞态关系，不同的分段锁之间没有锁竞争。

相比于对整个Map加锁的设计，分段锁大大的提高了高并发环境下的处理能力。但同时，由于不是对整个Map加锁，导致一些需要扫描整个Map的方法（如size(), containsValue()）需要使用特殊的实现，另外一些方法（如clear()）甚至放弃了对一致性的要求（ConcurrentHashMap是弱一致性的）。

ConcurrentHashMap中的HashEntry相对于HashMap中的Entry有一定的差异性：CHM的HashEntry中的value以及next都被volatile修饰，这样在多线程读写过程中能够保持它们的可见性，代码如下：

    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;

Hashtable虽然性能上不如ConcurrentHashMap，但并不能完全被取代，两者的迭代器的一致性不同的，Hashtable的迭代器是强一致性的，而ConcurrentHashMap是弱一致的。 ConcurrentHashMap的 get，clear，iterator 都是弱一致性的。

get方法是弱一致的，是什么含义？可能你期望往ConcurrentHashMap底层数据结构中加入一个元素后，立马能对get可见，但ConcurrentHashMap并不能如你所愿。换句话说，put操作将一个元素加入到底层数据结构后，get可能在某段时间内还看不到这个元素，若不考虑内存模型，单从代码逻辑上来看，却是应该可以看得到的。

#### 3.1.2 弱一致性解释

>[https://my.oschina.net/hosee/blog/675423](https://my.oschina.net/hosee/blog/675423)

Segment#put

    V put(K key, int hash, V value, boolean onlyIfAbsent) {
        lock();
        try {
            int c = count;
            if (c++ > threshold) // ensure capacity
                rehash();
            HashEntry[] tab = table;
            int index = hash & (tab.length - 1);
            HashEntry first = tab[index];
            HashEntry e = first;
            while (e != null && (e.hash != hash || !key.equals(e.key)))
                e = e.next;

            V oldValue;
            if (e != null) {
                oldValue = e.value;
                if (!onlyIfAbsent)
                    e.value = value;
            }
            else {
                oldValue = null;
                ++modCount;
                tab[index] = new HashEntry(key, hash, first, value);
                count = c; // write-volatile
            }
            return oldValue;
        } finally {
            unlock();
        }
    }

Segment#get

    V get(Object key, int hash) {
        if (count != 0) { // read-volatile
            HashEntry e = getFirst(hash);
            while (e != null) {
                if (e.hash == hash && key.equals(e.key)) {
                    V v = e.value;
                    if (v != null)
                        return v;
                    return readValueUnderLock(e); // recheck
                }
                e = e.next;
            }
        }
        return null;
    }


同一个Segment实例中的put操作是加了锁的，而对应的get却没有。

put操作可以分为两种情况，一是key已经存在，修改对应的value；二是key不存在，将一个新的Entry加入底层数据结构。

- key已经存在的情况比较简单，即if (e != null)部分，前面已经说过HashEntry的value是个volatile变量，当线程1给value赋值后，会立马对执行get的线程2可见，而不用等到put方法结束。
- key不存在的情况稍微复杂一些，新加一个Entry的逻辑在else中。那么将new HashEntry赋值给tab[index]是否能立刻对执行get的线程可见呢？答案是否定的。

除了get是弱一致性，clear()函数也是。因为没有全局的锁，在清除完一个segments之后，正在清理下一个segments的时候，已经清理segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。

ConcurrentHashMap的弱一致性主要是为了提升效率，是一致性与效率之间的一种权衡。要成为强一致性，就得到处使用锁，甚至是全局锁，这就与Hashtable和同步的HashMap一样了。

#### 3.1.3 并发度
并发度可以理解为程序运行时能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数，实际上就是ConcurrentHashMap中的分段锁个数，即Segment[]的数组长度。 默认16，可以自行设置。

如果并发度设置的过小，会带来严重的锁竞争问题；如果并发度设置的过大，原本位于同一个Segment内的访问会扩散到不同的Segment中，CPU cache命中率会下降，从而引起程序性能下降。

#### 3.1.4 创建分段锁
JDK7中除了第一个Segment之外，剩余的Segments采用的是延迟初始化的机制：每次put之前都需要检查key对应的Segment是否为null，如果是则调用ensureSegment()以确保对应的Segment被创建。

#### 3.1.5 put/putIfAbsent/putAll

put()会用tryLock()来获取锁。JDK7做出的优化：在真正申请锁之前，put方法会通过tryLock()方法尝试获得锁，在尝试获得锁的过程中会对对应hashcode的链表进行遍历，如果遍历完毕仍然找不到与key相同的HashEntry节点，则为后续的put操作提前创建一个HashEntry。当tryLock一定次数后仍无法获得锁，则通过lock申请锁。

在获得锁之后，Segment对链表进行遍历，如果某个HashEntry节点具有相同的key，则更新该HashEntry的value值，否则新建一个HashEntry节点，将它设置为链表的新head节点并将原头节点设为新head的下一个节点。新建过程中如果节点总数（含新建的HashEntry）超过threshold，则调用rehash()方法对Segment进行扩容，最后将新建HashEntry写入到数组中。

put方法中，链接新节点的下一个节点（HashEntry.setNext()）以及将链表写入到数组中（setEntryAt()）都是通过Unsafe的putOrderedObject()方法来实现，这里并未使用具有原子写语义的putObjectVolatile()的原因是：JMM会保证获得锁到释放锁之间所有对象的状态更新都会在锁被释放之后更新到主存，从而保证这些变更对其他线程是可见的。

#### 3.1.5 rehash
相对于HashMap的resize。 但有优化:避免让所有的节点都进行复制操作：由于扩容是基于2的幂指来操作，假设扩容前某HashEntry对应到Segment中数组的index为i，数组的容量为capacity，那么扩容后该HashEntry对应到新数组中的index只可能为i或者i+capacity，因此大多数HashEntry节点在扩容前后index可以保持不变。

基于此，rehash方法中会定位第一个后续所有节点在扩容后index都保持不变的节点，然后将这个节点之前的所有节点重排即可。经过rehash之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

JDK1.7 使用的是单链表的头插入方式，如果同个位置发生hash碰撞，原来位置的节点会不断往后移动。每次扩容后，顺序会倒。
（红黑树是在jdk1.8后引入的，之前的版本的HashEntry还是数组。而且jdk1.8不会像1.7一样再次计算hash，而且jdk1.8扩容的顺序不会再变。）

## 四、CHM支持完全并发的读
ConcurrentHashMap完全允许多个读操作并发进行，读操作并不需要加锁。（事实上，ConcurrentHashMap支持完全并发的读以及一定程度并发的写。）

    static final class HashEntry<K,V> {  
        final K key;  
        final int hash;  
        volatile V value;  
        final HashEntry<K,V> next;  
  
        HashEntry(K key, int hash, HashEntry<K,V> next, V value) {  
            this.key = key;  
            this.hash = hash;  
            this.next = next;  
            this.value = value;  
        }  
  
        @SuppressWarnings("unchecked")  
        static final <K,V> HashEntry<K,V>[] newArray(int i) {  
            return new HashEntry[i];  
        }  
    }  

可以看到除了value不是final的，其它值都是final的，这意味着不能从hash链的中间或尾部添加或删除节点，因为这需要修改next引用值，所有的节点的修改只能从头部开始。

**对于put操作，可以一律添加到Hash链的头部。但是对于remove操作，可能需要从中间删除一个节点，这就需要将要删除节点的前面所有节点整个复制一遍，最后一个节点指向要删除结点的下一个结点。**为了确保读操作能够看到最新的值，将value设置成volatile，这避免了加锁。 

remove操作要注意一个问题：如果某个读操作在删除时已经定位到了旧的链表上，那么此操作仍然将能读到数据，只不过读取到的是旧数据而已，这在多线程里面是没有问题的（也算是一个弱一致性问题）。

**在 ConcurrentHashMap 中，不允许用 null作为键和值**，当读线程读到某个 HashEntry 的 value 域的值为 null 时，便知道产生了冲突——发生了重排序现象，需要加锁后重新读入这个 value 值。这些特性互相配合，使得读线程即使在不加锁状态下，也能正确访问 ConcurrentHashMap。 

## REF

>[http://www.importnew.com/26049.html](http://www.importnew.com/26049.html)
>[https://www.cnblogs.com/study-everyday/p/6430462.html](https://www.cnblogs.com/study-everyday/p/6430462.html)  
>[https://www.cnblogs.com/slwenyi/p/6393829.html](https://www.cnblogs.com/slwenyi/p/6393829.html)










