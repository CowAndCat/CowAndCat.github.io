---
layout: post
title: Redis底层数据存储
category: redis
comments: false
---

## 一、概述
Redis 数据库里面的每个键值对（key-value） 都是由对象（object）组成的：

- 数据库键总是一个字符串对象（string object）;
- 数据库的值则可以是字符串对象、列表对象（list）、哈希对象（hash）、集合对象（set）、有序集合（sort set）对象这五种对象中的其中一种。

实现总结：

- string: 带有len和free的SDS
- list：双向链表/压缩列表ziplist
- hash：基于双hashtable的字典/压缩列表
- 集合对象：存int的Intset/hashtable
- 有序集合：多层skiplist/ziplist

Redis底层数据结构共有八种：

|编码常量  |  编码所对应的底层数据结构|
|--|--|
|REDIS_ENCODING_INT  |long 类型的整数|
|REDIS_ENCODING_EMBSTR  | embstr 编码的简单动态字符串|
|REDIS_ENCODING_RAW  |简单动态字符串|
|REDIS_ENCODING_HT   |字典|
|REDIS_ENCODING_LINKEDLIST |  双端链表|
|REDIS_ENCODING_ZIPLIST  |压缩列表|
|REDIS_ENCODING_INTSET   |整数集合|
|REDIS_ENCODING_SKIPLIST |跳跃表和字典|

## 二、简单动态字符串（SDS, simple dynamic string）
Redis 是一个开源的使用ANSI C语言编写的key-value 数据库,但Redis 没有直接使用C语言传统的字符串表示，而是自己构建了一种名为简单动态字符串（simple dynamic string SDS）的抽象类型，并将SDS用作Redis 的默认字符串表示：

    redis>SET msg "hello world"
    OK

设置一个key= msg，value = hello world 的新键值对，他们底层是数据结构将会是：

- 键（key）是一个字符串对象，对象的底层实现是一个保存着字符串“msg” 的SDS；
- 值（value）也是一个字符串对象，对象的底层实现是一个保存着字符串“hello world” 的SDS

从上述例子，我们可以很直观的看到我们在平常使用redis 的时候，创建的字符串到底是一个什么样子的数据类型。除了用来保存字符串以外，SDS还被用作缓冲区（buffer）AOF模块中的AOF缓冲区。

### 2.1 SDS定义
Redis 中定义动态字符串的结构：

    /*  
     * 保存字符串对象的结构  
     */  
    struct sdshdr {  
          
        // buf 中已占用空间的长度  
        int len;  
      
        // buf 中剩余可用空间的长度  
        int free;  
      
        // 数据空间  
        char buf[];  
    }; 

### 2.2  SDS 与 C 字符串的区别
- 二进制安全，可以保存二进制数据和文本文数据  
C 字符串中的字符必须符合某种编码，并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些限制使得C字符串只能保存文本数据，而不能保存想图片，音频，视频，压缩文件这样的二进制数据。  
但是在Redis中，不是靠空字符来判断字符串的结束的，而是通过len这个属性。那么，即便是中间出现了空字符对于SDS来说，读取该字符仍然是可以的。

- 获取字符串长度（SDS O(1) /C 字符串 O(n)）  
传统的C字符串 使用长度为N+1 的字符串数组来表示长度为N 的字符串，所以为了获取一个长度为C字符串的长度，必须遍历整个字符串。SDS只要获取len值。

- 杜绝缓冲区溢出  
C 字符串 不记录字符串长度，除了获取的时候复杂度高以外，还容易导致缓冲区溢出。  
SDS 的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当我们需要对一个SDS 进行修改的时候，redis 会在执行拼接操作之前，预先检查给定SDS 空间是否足够，如果不够，会先拓展SDS 的空间，然后再执行拼接操作。

- 减少修改字符串时带来的内存重分配次数  
C语言字符串在进行字符串的扩充和收缩的时候，都会面临着内存空间的重新分配问题。  
1.字符串拼接会产生字符串的内存空间的扩充，在拼接的过程中，原来的字符串的大小很可能小于拼接后的字符串的大小，那么这样的话，就会导致一旦忘记申请分配空间，就会导致内存的溢出。  
2.字符串在进行收缩的时候，内存空间会相应的收缩，而如果在进行字符串的切割的时候，没有对内存的空间进行一个重新分配，那么这部分多出来的空间就成为了内存泄露。
SDS的free字段能避免重新分配：如果在上一次修改字符串的时候已经拓展了空间，再次进行修改字符串的时候会发现空间足够使用，因此无须进行空间拓展；SDS通过预分配策略将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次

- 惰性空间释放  
在对字符串进行收缩的时候，我们也可以使用free 属性来进行记录剩余空间，这样做的好处就是避免下次对字符串进行再次修改的时候，需要对字符串的空间进行拓展。

## 三、链表
链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

### 3.1 链表的数据结构
每个链表节点使用一个 listNode结构表示（adlist.h/listNode）：

    typedef struct listNode {
        struct listNode *prev;
        struct listNode *next;
        void * value;  
    }

    typedef struct list {        
        listNode  * head; //表头节点        
        listNode  * tail; //表尾节点        
        unsigned long len; //链表长度        
        void *(*dup) (void *ptr); //节点值复制函数        
        void (*free) (void *ptr); //节点值释放函数        
        int (*match)(void *ptr, void *key); //节点值对比函数
    }

链表的特性
- 双端：链表节点带有prev 和next 指针，获取某个节点的前置节点和后置节点的时间复杂度都是O（N）
- 无环：表头节点的 prev 指针和表尾节点的next 都指向NULL，对立案表的访问时以NULL为截止
- 表头和表尾：因为链表带有head指针和tail 指针，程序获取链表头结点和尾节点的时间复杂度为O(1)
- 长度计数器：链表中存有记录链表长度的属性 len
- 多态：链表节点使用 void* 指针来保存节点值，并且可以通过list 结构的dup 、 free、 match三个属性为节点值设置类型特定函数。

## 四、字典
字典，又称为符号表（symbol table）、关联数组（associative array）或映射（map），是一种用于保存键值对的抽象数据结构。

### 4.1 哈希表
    typeof struct dictEntry {
       //键
       void *key;
       //值
       union {
          void *val;
          uint64_tu64;
          int64_ts64;
       }
       struct dictEntry *next;
    }

    typedef struct dictht {
       //哈希表数组
       dictEntry **table;
       //哈希表大小
       unsigned long size;
       //哈希表大小掩码，用于计算索引值
       unsigned long sizemask;
       //该哈希表已有节点的数量
       unsigned long used;
    }

    typedef struct dict {
        // 类型特定函数
        dictType *type;
        // 私有数据
        void *privedata;
        // 哈希表
        dictht  ht[2];
        // rehash 索引
        in trehashidx;

    }

在dictht结构中存有指向dictEntry 数组的指针，dictEntry是用来存储数据的空间。

存入里面的key 并不是直接的字符串，而是一个hash 值。如果发生hash碰撞，Redis 采用链地址法解决。

dict里的type 属性 和privdata 属性是针对不同类型的键值对，为创建多态字典而设置的。
ht 属性是一个包含两个项（两个哈希表）的数组。

### 4.2 Rehash
随着对哈希表的不断操作，哈希表保存的键值对会逐渐的发生改变，为了让哈希表的负载因子维持在一个合理的范围之内，我们需要对哈希表的大小进行相应的扩展或者压缩，这时候，我们可以通过 rehash（重新散列）操作来完成。

size为2的ht在这时发挥作用，哈希表空间分配规则：

- 如果执行的是拓展操作，那么ht[1] 的大小为第一个大于等于ht[0] 的2的n次幂
- 如果执行的是收缩操作，那么ht[1] 的大小为第一个大于等于ht[0] 的2的n次幂

数据迁移：将ht[0]中的数据转移到ht[1]中，在转移的过程中，需要对哈希表节点的数据重新进行哈希值计算。

释放ht[0]：将ht[0]释放，然后将ht[1]设置成ht[0]，最后为ht[1]分配一个空白哈希表：

如果ht[0]的数据量很大怎么办？  
渐进式rehash 的详细步骤：

- 1、为ht[1] 分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
- 2、在几点钟维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash 开始
- 3、在rehash 进行期间，每次对字典执行CRUD操作时，程序除了执行指定的操作以外，还会将ht[0]中的数据rehash 到ht[1]表中，并且将rehashidx加一
- 4、当ht[0]中所有数据转移到ht[1]中时，将rehashidx 设置成-1，表示rehash 结束

采用渐进式rehash 的好处在于它采取分而治之的方式，避免了集中式rehash 带来的庞大计算量。

## 五、跳表

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。跳跃表是一种随机化的数据,跳跃表以有序的方式在层次化的链表中保存元素，效率和平衡树媲美 ——查找、删除、添加等操作都可以在对数期望时间下完成，并且比起平衡树来说，跳跃表的实现要简单直观得多。

Redis 只在两个地方用到了跳跃表，一个是实现有序集合键，另外一个是在集群节点中用作内部数据结构。

### 5.1 zskiplistNode（节点） 数据结构

    typedef struct zskiplistNode {
        //层
        struct zskiplistLevel{
            //前进指针
            struct zskiplistNode *forward;
            //跨度
            unsigned int span;
        } level[];

    　　 //后退指针
        struct zskiplistNode *backward;
    　　 //分值
        double score;
    　　 //成员对象
        robj *obj;
    }

    typedef struct zskiplist {
         //表头节点和表尾节点
         structz skiplistNode *header,*tail;
         //表中节点数量
         unsigned long length;
         //表中层数最大的节点的层数
         int level;

    } zskiplist;

zskiplistNode：  
1、层：level 数组可以包含多个元素，每个元素都包含一个指向其他节点的指针。  
2、前进指针：用于指向表尾方向的前进指针  
3、跨度：用于记录两个节点之间的距离  
4、后退指针：用于从表尾向表头方向访问节点  
5、分值和成员：跳跃表中的所有节点都按分值从小到大排序。成员对象指向一个字符串，这个字符串对象保存着一个SDS值

### 5.2 总结

- 跳跃表是有序集合的底层实现之一
- 主要有zskiplist 和zskiplistNode两个结构组成
- 每个跳跃表节点的层高都是1至32之间的随机数
- 在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的对象必须是唯一的
- 节点按照分值的大小从大到小排序，如果分值相同，则按成员对象大小排序

## 六、整数集合（Intset）

整数集合是集合建的底层实现之一，当一个集合中只包含整数，且这个集合中的元素数量不多时，redis就会使用整数集合intset作为集合的底层实现。它是一个特殊的集合，里面存储的数据只能够是整数，并且数据量不能过大。


### 6.1 数据结构

    typedef struct intset{
        //编码方式
        uint32_t enconding;
        // 集合包含的元素数量
        uint32_t length;
        //保存元素的数组    
        int8_t contents[];
    }  

intset将数组定义为int8_t，但实际上数组保存的元素类型取决于encoding

### 6.2 Intset升级
Intset 中升级整数集合（当我们存入的整数不符合整数集合中的编码格式时）并添加新元素共分为三步进行：

- 1、根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
- 2、将底层数组现有的所有元素都转换成新的编码格式，重新分配空间
- 3、将新元素加入到底层数组中

整数集合升级的好处：1、提升灵活性 2、节约内存

整数集合只支持升级操作，不支持降级操作.

## 七、压缩列表
压缩列表是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数，要么就是长度比较短的字符串，那么Redis 就会使用压缩列表来做列表键的底层实现。

### 7.1 构成

![组成](/images/201901/compresslist1.png)

1、zlbytes:用于记录整个压缩列表占用的内存字节数  
2、zltail：记录要列表尾节点距离压缩列表的起始地址有多少字节  
3、zllen：记录了压缩列表包含的节点数量。  
4、entryX：要说列表包含的各个节点  
5、zlend：用于标记压缩列表的末端  

![组成](/images/201901/compresslist2.png)

### 7.2 总结

- 压缩列表是一种为了节约内存而开发的顺序型数据结构
- **压缩列表被用作列表键和哈希键的底层实现之一**
- 压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值
- 添加新节点到压缩列表，可能会引发连锁更新操作。



## REF
> [深入浅出Redis-redis底层数据结构（上）](http://www.cnblogs.com/jaycekon/p/6227442.html)  
> [深入浅出Redis-redis底层数据结构（下）](https://www.cnblogs.com/jaycekon/p/6277653.html)  
> [redis支持的五种数据类型及其底层实现](https://blog.csdn.net/u010426961/article/details/78802336)