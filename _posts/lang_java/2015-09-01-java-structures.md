---
layout: post
title: Java中的集合类Collection，List, Set, Map; Array, Stack，Queue等。
category: java
comments: false
---
## Java中的主要数据结构（整体图）
首先，要对这堆数据结构有一个大致的认识：

![Java Data Structure](/images/201509/javaStructures.bmp "Java Data Structure")

或者下列的树结构：

		Collection  
		├List  
		│├LinkedList   
		│├ArrayList  
		│└Vector  
		│　└Stack  
		└Set  

		Map  
		├Hashtable  
		├HashMap  
		├WeakHashMap  
		└TreeMap

## 经验总结

* 如果涉及到堆栈，队列等操作，应该考虑用List，对于需要快速插入，删除元素，应该使用LinkedList，如果需要快速随机访问元素，应该使用ArrayList。
* 如果程序在单线程环境中，或者访问仅仅在一个线程中进行，考虑非同步的类，其效率较高，如果多个线程可能同时操作一个类，应该使用同步的类。
* 要特别注意对哈希表的操作，作为key的对象要正确复写equals和hashCode方法。
* 尽量返回接口而非实际的类型，如返回List而非ArrayList，这样如果以后需要将ArrayList换成LinkedList时，客户端代码不用改变。这就是针对抽象编程。

------------------

## Collection
List和Set的父类。  
**注意**：Java SDK不提供直接继承自Collection的类，Java SDK提供的类都是继承自Collection的“子接口”如List和Set。

### 内置函数(工具类collections)
工具类collections用于操作集合类（[collections在java中的常见用法](http://blog.sina.com.cn/s/blog_a46817ff01017hqr.html)),如List,Set,常用方法有:

1. 排序（Sort）  
	使用sort方法可以根据元素的自然顺序,对指定列表按**升序**进行排序。  

	列表中的所有元素都必须实现 Comparable 接口。此列表内的所有元素都必须是使用指定比较器可相互比较的。如：

		double array[] = {112, 111, 23, 456, 231 };
		for (int i = 0; i < array.length; i++) {
		list.add(new Double(array[i]));
		}

		Collections.sort(list);

		for (int i = 0; i < array.length; i++) {
		System.out.println(li.get(i));
		}
		//结果：112,111,23,456,231

2. 混排（Shuffling）  
	混排算法所做的正好与 sort 相反： 它打乱在一个 List 中可能有的任何排列的踪迹。也就是说，基于随机源的输入重排该 List，这样的排列具有相同的可能性（假设随机源是公正的）。

		double array[] = {112, 111, 23, 456, 231 };
		for (int i = 0; i < array.length; i++) {
		list.add(new Double(array[i]));
		}

		Collections.shuffle(list);

		for (int i = 0; i < array.length; i++) {
		System.out.println(li.get(i));
		}
		//结果：112,111,23,456,231

3. 反转（Reverse）  
	使用Reverse方法可以根据元素的自然顺序 对指定列表按降序进行排序。

		double array[] = {112, 111, 23, 456, 231 };
		for (int i = 0; i < array.length; i++) {
			list.add(new Double(array[i]));
		}

		Collections.reverse (list);

		for (int i = 0; i < array.length; i++) {
			System.out.println(li.get(i));
		}
		//结果：231,456,23,111,112

4. 替换所有的元素（Fill）  
	使用指定元素替换指定列表中的所有元素。

		String str[] = {"dd","aa","bb","cc","ee"};
		for(int j=0;j<str.length(); j++){
			li.add(new String(str[j]));
		}

		Collections.fill(li,"aaa");

		for (int i = 0; i < li.size(); i++) {
			System.out.println("list[" + i + "]=" + li.get(i));
		}
		//结果：aaa,aaa,aaa,aaa,aaa
5. 拷贝（Copy）  
	用两个参数，一个目标 List 和一个源 List， 将源的元素拷贝到目标，并覆盖它的内容。目标 List 至少与源一样长。如果它更长，则在目标 List 中的剩余元素不受影响。  

	Collections.copy(list,li); //后面一个参数是目标列表 ，前一个是源列表

6. 返回Collections中最小元素（min）和最大元素

	Collections.min(list);  
	Collections.max(list);
7. Rotate  
	根据指定的距离循环移动指定列表中的元素
	Collections.rotate（list，-1）；
	如果是负数，则正向移动，正数则方向移动

8. static int binarySearch(List list,Object key)  
	使用二分搜索查找key对象的索引值，因为使用的二分查找，所以前提是必须有序。

9. static int frequency(Collection c,Object o)  
	返回指定元素在集合中出现在次数


---------------------------

## List

### 基本概念  

* List是有序的Collection。类似Java的数组。  
* 和set相比，允许有相同的元素。  
* 除了具有Collection接口必备的iterator()方法外，List还提供一个listIterator()方法，返回一个 ListIterator接口，和标准的Iterator接口相比，ListIterator多了一些add()之类的方法，允许添加，删除，设定元素， 还能向前或向后遍历。ListIterator：

		List<Integer> list = new LinkedList<Integer>();
		ListIterator iter = list.listIterator();
		iter.add(element);
		iter.next();
		iter.hasNext();
		iter.previous();
		iter.hasPrevious();
		iter.remove();
		iter.set(element);

### 实现类：

1. LinkedList：  
  - 允许null元素。  
  - 此外LinkedList提供额外的get，remove，insert（list.add(index, element);list.set(index, element);）方法在 LinkedList的首部或尾部。这些操作使LinkedList可被用作堆栈（stack），队列（queue）或双向队列（deque）。
  - 注意LinkedList没有同步方法。如果多个线程同时访问一个List，则必须自己实现访问同步。一种解决方法是在创建List时构造一个同步的List：  

		List list = Collections.synchronizedList(new LinkedList(...));

2. ArrayList
	- ArrayList实现了**可变大小**的数组。
	- 它允许所有元素，包括null。
	- ArrayList没有同步。
	- size，isEmpty，get，set方法运行时间为常数。但是add方法开销为分摊的常数，添加n个元素需要O(n)的时间。其他的方法运行时间为线性。

3. Vector
	- Vector非常类似ArrayList，但是Vector是同步的。

4. Stack
	- Stack继承自Vector，实现一个后进先出的堆栈。
	- Stack提供5个额外的方法使得Vector得以被当作堆栈使用。基本的push和pop方法，还有peek方法得到栈顶的元素，empty方法测试堆栈是否为空，search方法检测一个元素在堆栈中的位置。

### 相互区别
* Vector和ArrayList
 1. Vector是线程同步的，所以它也是线程安全的，而arraylist是线程异步的，是不安全的。如果不考虑到线程的安全因素，一般用arraylist效率比较高。
 2. 如果集合中的元素的数目大于目前集合数组的长度时，vector增长率为目前数组长度的100%,而arraylist增长率为目前数组长度的50%.如过在集合中使用数据量比较大的数据，用vector有一定的优势。
 3. Vector由于使用了synchronized方法（线程安全）所以性能上比ArrayList要差，LinkedList使用双向链表实现存储，按序号索引数据需要进行向前或向后遍历，但是插入数据时只需要记录本项的前后项即可，所以插入数度较快

* arraylist和linkedlist
	1. ArrayList是实现了基于动态数组的数据结构，LinkedList基于链表的数据结构。
	2. 对于随机访问get和set，ArrayList觉得优于LinkedList，因为LinkedList要移动指针。
	3. 对于新增和删除操作add和remove，LinedList比较占优势，因为ArrayList要移动数据。


---------------------------

## Set

### 基本概念
Set是一种不包含重复的元素的Collection，即任意的两个元素e1和e2都有e1.equals(e2)=false，Set最多有一个null元素。

### 实现类型：
1. TreeSet
 	- 是Set的一种变体——可以实现排序等功能的集合。（TreeSet类实现了SortedSet接口，能够对集合中的对象进行排序。）

			treeSet.higher(element);//	Returns the least element in this set strictly greater than the given element, or null if there is no such element.  

			treeSet.lower(element);

			treeSet.floor(element);//<=element的最大数字

			treeSet.descendingSet();//Returns a reverse order view of the elements contained in this set.

2. HashSet
	- 允许包含值为null的元素，但最多只能有一个null元素。

### 常用方法
		set.add(element);
		set.remove(element);
		set.size();
		set.isEmpty();

### 遍历
		Set<String> set = new HashSet<String>();
		//iteration
		Iterator<String> it = set.iterator();  
		while (it.hasNext()) {  
		  String str = it.next();  
		  System.out.println(str);  
		}  

		//for循环遍历：  
		for (String str : set) {  
		      System.out.println(str);  
		}  


---------------------------

## Map

### 基本概念
Map没有继承Collection接口，Map提供key到value的映射。

### 实现类型

1. HashMap
 - HashMap和Hashtable类似，不同之处在于HashMap是非同步的；
 - 允许null，即null value和null key。(只有HashMap可以让你将空值作为一个表的条目的key或value)
 - 但是将HashMap视为Collection时（values()方法可返回Collection），其迭代子操作时间开销和HashMap 的容量成比例。因此，如果迭代操作的性能相当重要的话，不要将HashMap的初始化容量设得过高，或者load factor过低。


2. WeakHashMap
	- WeakHashMap是一种改进的HashMap，它对key实行“弱引用”，如果一个key不再被外部所引用，那么该key可以被GC回收。

3. Hashtable
  - 实现一个key-value映射的哈希表。任何非空（non-null）的对象都可作为key或者value。
  - Hashtable通过initial capacity和load factor两个参数调整性能。通常缺省的load factor 0.75较好地实现了时间和空间的均衡。增大load factor可以节省空间但相应的查找时间将增大，这会影响像get和put这样的操作。
  - 由于作为key的对象将通过计算其散列函数来确定与之对应的value的位置，因此任何作为key的对象都必须实现hashCode和equals方法。
  - Hashtable是同步的。
4. TreeMap

### 相互区别
* HashMap与TreeMap
	1. HashMap通过hashcode对其内容进行快速查找，而TreeMap中所有的元素都保持着某种固定的顺序，如果你需要得到一个有序的结果你就应该使用TreeMap（HashMap中元素的排列顺序是不固定的）。
	2. 在Map 中插入、删除和定位元素，HashMap 是最好的选择。但如果您要按自然顺序或自定义顺序遍历键，那么TreeMap会更好。使用HashMap要求添加的键类明确定义了hashCode()和 equals()的实现。　　
	3. TreeMap没有调优选项，因为该树总处于平衡状态。

### 常用操作：

	map.put(key, value);
	map.isEmpty();
	map.get(key);
	map.isEmpty();
	map.clear();
	map.containsKey(key);
	map.containsValue(value);
	map.remove(key);
	map.size();


### 遍历方式：  

	Map<String, String> map = new HashMap<String, String>();
    map.put("1", "a");
    map.put("2", "b");
    map.put("3", "c");

    //最简洁、最通用的遍历方式
    for (Map.Entry<String, String> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " = " + entry.getValue());
    }

    //方式2
    for (Iterator<Map.Entry<String, String>> it = map.entrySet().iterator(); it.hasNext();) {
            Map.Entry<String, String> entry = it.next();
            System.out.println(entry.getKey() + " = " + entry.getValue());
    }

    //方式3
    for (Iterator<String> it = map.keySet().iterator(); it.hasNext();) {
            String key = it.next();
            System.out.println(key + " = " + map.get(key));
    }

---------------------------

## Queue

### 实现类型：

1. PriorityQueue <T\>
2. LinkedList
3. LinkedBlockingQueue <T\>  
 LinkedBlockingQueue实现是线程安全的，实现了先进先出等特性，是作为生产者消费者的首选，LinkedBlockingQueue 可以指定容量，也可以不指定，不指定的话，默认最大是Integer.MAX_VALUE，其中主要用到put和take方法，put方法在队列满的时候会阻塞直到有队列成员被消费，take方法在队列空的时候会阻塞，直到有队列成员被放进来。
4. ConcurrentLinkedQueue  
ConcurrentLinkedQueue是Queue的一个安全实现．Queue中元素按FIFO原则进行排序．采用CAS操作，来保证元素的一致性。

普通的LinkedList实现并没有定义特殊的排序算法，所以输出元素时会按照插入的顺序.

PriorityQueue的内部是一个min heap,会按照从上至下，从左至右的输出。

[并发队列ConcurrentLinkedQueue和阻塞队列LinkedBlockingQueue用法](http://www.cnblogs.com/linjiqin/archive/2013/05/30/3108188.html)

### Queue Interface Structure  
		       Throws exception	Returns special value  
		Insert	add(e)	        offer(e)  
		Remove	remove()	    poll()  
		Examine	element()	    peek()
remove和poll方法都删除并返回Queue中的头元素（注意，并不是插入的第一个元素，因为有的Queue实现是排序的）。当Queue为空时，remove抛出NoSuchElementException异常，而poll返回null。

element和peek返回但不删除Queue中的头元素，它们的区别类似remove与poll。

### 常用方法

	queue.add(element); //注意：没有push方法或put方法
	queue.offer(element);///与add的区别：
	//1. add属于interface Collection<E>，而offer属于interface Deque<E>；
	//2. When using a capacity-restricted queue, this method is generally preferable to add, which can fail to insert an element only by throwing an exception.（如果队列满了，add会抛异常，而offer返回false.

	queue.poll();		//注意：没有pop方法
	queue.peek();
	queue.clear();
	queue.containsAll(collection); //是否包含集合中的所有元素
	queue.retainAll(collection); //保留集合中的元素
	queue.removeAll(collection); //删除集合中的元素
	queue.toArray()；

### 遍历

	//集合方式遍历，元素不会被移除
    for (Integer element : queue) {
            System.out.println(element);
    }

    //队列方式遍历，元素逐个被移除
    while (queue.peek() != null) {
            System.out.println(queue.poll());
    }


---------------------------

## Stack

### 初始化类型：
Stack

### 常用方法：

	stack.push(element);
	stack.peek();
	stack.pop();
	stack.containsAll(collection); //是否包含集合中的所有元素
	stack.retainAll(collection); //保留集合中的元素
	stack.removeAll(collection); //删除集合中的元素
	stack.toArray();
	stack.copyInto(anArray);

Java中的stack都不是严格意义上的stack了，例如可以：  

	stack.add(index, element);
	stack.get(index);
	stack.lastElement()；
	stack.firstElement()；
	stack.set(index, element);
	stack.setElementAt(element, index);
	stack.elementAt(index);    //返回element类型

### 遍历方法：

	Stack<Integer> s = new Stack<Integer>();
    for (int i = 0; i < 10; i++) {
            s.push(i);
    }
    //集合遍历方式
    for (Integer element : s) {
        System.out.println(element);
    }

	//栈弹出遍历方式
	//while (s.peek()!=null) {
    //不健壮的判断方式，容易抛异常，正确写法是下面的
    while (!s.empty()) {
		System.out.println(s.pop());
    }

---------------

## Arrays
Java 2 在 java.util 中新增加了一个叫做 Arrays 的类。这个类提供了各种在进行数组运算 时很有用的方法。尽管这些方法在技术上不属于类集框架，但它们提供了跨越类集和数组的桥梁。

## 常见方法
- static List asList(Object[ ] array) //视为list
- binarySearch()方法使用二分法搜索寻找指定的值。该方法必须应用于排序数组。如：

		static int binarySearch(int[ ] array, int value)

---------------

## Enumeration
Enumeration(更像是数据类型） 接口定义了可以对一个对象的类集中的元素进行枚举（一次获得一个）的 方法。这个接口尽管没有被摈弃，但已经被 Iterator 所替代。  

Enumeration 指定下面的两个方法：  

  -  boolean hasMoreElements()   
  - Object nextElement()

执行后，当仍有更多的元素可提取时，hasMoreElements()方法一定返回 true。当所有元素都被枚举了，则返回 false。

nextElement()方法将枚举中的下一个对象做为一个类属 Object
的引用而返回。也就是每次调用 nextElement()方法获得枚举中的下一个对象。调用例程必须
将那个对象转换为包含在枚举内的对象类型.

------------

## Properties
属性（Properties）是 Hashtable 的一个子类。它用来保持值的列表，在其中关键字和值 都是字符串（String）。

Properties 类被许多其他的Java 类所使用。例如，当获得系统环境 值时，System.getProperties()返回对象的类型。

Properties 类的一个有用的功能是可以指定一个默认属性，如果没有值与特定的关键字 相关联，则返回这个默认属性。例如，默认值可以与关键字一起在 getProperty()方法中被指
定——如 getProperty(“name”，“default value”)。如果“name”值没有找到，则返回“default value”。

--------------------

## Dictionary
Dictionary 类是一个抽象类，用来存储键/值对，作用和Map类相似。

给出键和值，你就可以将值存储在Dictionary对象中。一旦该值被存储，就可以通过它的键来获取它。所以和Map一样， Dictionary 也可以作为一个键/值对列表.

Dictionary类已经过时了。在实际开发中，你可以实现Map接口来获取键/值的存储功能。

-------------

## BitSet（位集合）
位集合类实现了一组可以单独设置和清除的位或标志。

该类在处理一组布尔值的时候非常有用，你只需要给每个值赋值一"位"，然后对位进行适当的设置或清除，就可以对布尔值进行操作了。

一个Bitset类创建一种特殊类型的数组来保存位值。BitSet中数组大小会随需要增加。这和位向量（vector of bits）比较类似。

详细点击:[http://www.runoob.com/java/java-bitset-class.html](http://www.runoob.com/java/java-bitset-class.html)

------------------------

## 常见面试题
1. ArrayList和Vector有什么区别？HashMap和HashTable有什么区别？  
A: Vector和HashTable是线程同步的（synchronized）。性能上，ArrayList和HashMap分别比Vector和Hashtable要好。

2. 大致讲解java集合的体系结构  
	List、Set、Map是这个集合体系中最主要的三个接口。  
      其中List和Set继承自Collection接口。  
      Set不允许元素重复。HashSet和TreeSet是两个主要的实现类  。
      List有序且允许元素重复。ArrayList、LinkedList和Vector是三个主要的实现类。  
      Map也属于集合系统，但和Collection接口不同。Map是key对value的映射集合，其中key列就是一个集合。key不能重复，但是value可以重复。HashMap、TreeMap和Hashtable是三个主要的实现类。  
      SortedSet和SortedMap接口对元素按指定规则排序，SortedMap是对key列进行排序。

3. Comparable和Comparator区别  
 调用java.util.Collections.sort(List list)方法来进行排序的时候，List内的Object都必须实现了Comparable接口。
 但如果是用java.util.Collections.sort(List list，Comparator c)，可以临时声明一个Comparator 来实现排序。

        Collections.sort(imageList, new Comparator() {
            public int compare(Object a, Object b) {
                int orderA = Integer.parseInt( ( (Image) a).getSequence());
                int orderB = Integer.parseInt( ( (Image) b).getSequence());
                return orderA - orderB;
           }
        });
4. stack和queue的相互实现  
	stack实现queue:用两个stack实现，一个管入队（enqueue），一个管出队（dequeue)；如果出队为空，将enqueue中的值放到dequeue中。  
	queue实现stack:两个queue，一个充当临时的队列，每次top或pop的时候，将存储的queue倒入到tmpqueue中，记录下最后一个，返回或删除，之后再倒回去。

### 参考
1. [Java集合的Stack、Queue、Map的遍历](http://lavasoft.blog.51cto.com/62575/181781/)  
2. [JAVA集合小结](http://www.blogjava.net/EvanLiu/archive/2007/11/12/159884.html)  
3. [Java集合类详解](http://blog.csdn.net/softwave/article/details/4166598)  
4. [java_Collection_介绍](http://blog.sina.com.cn/s/blog_3fb3625f0101aref.html)
