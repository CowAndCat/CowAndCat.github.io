---
layout: post
title: Java 面试题1
category: java
comments: false
---
## 1、代码阅读

### 1.1 下面代码的运行结果是？  


		class Person{
			public Person(){
				System.out.println("A person");
			}
		}

		public class Teacher extends Person {

			private String name="tom";

			public Teacher(){
				System.out.println("A teacher");
				super();
			}

			public static void main(String[] args) {
				Teacher teacher = new Teacher();
				System.out.println(this.name);
			}
		}

【答案】  
	 有两处错误：  
	 1. super()--Constructor call must be the first statement in a constructor.  
	 2. this--Cannot use this in a static context.

### 1.2 说出程序输出结果
	class X {
		Y y = new Y(); // 1

		X() {
			System.out.print("X");// 2
		}
	}

	class Y {
		Y() {
			System.out.print("Y");
		}
	}

	public class Z extends X {
		Y y = new Y();// 3

		Z() {
			System.out.print("Z");// 4
		}

		public static void main(String[] args) {
			new Z();
		}
	}

A: YXYZ,顺序已经在代码中标出。

	class A {
		public void print(){
			System.out.println("A");
		}
	}

	class B extends A{
		public void print(){
			System.out.println("B");
		}
	}

	public class Test{
		public static void main(String[]args){
			B objB=new B();
			objB.print();
			A objA=(A) objB;
			objA.print();
		}
	}

结果： B B
（objB本来定义为B）



### 1.3 有没有看过Java数据结构源代码？

 必须得回答看过啊。事先看。
 1. 先了解Set接口：

	public interface Set<E> extends Collection<E>{
		int size();
		boolean isEmpty();
		Iterator<E> iterator();
		//其他的函数
		boolean equals(Object o);
		int hashCode();
	}

  再看一个实现类，HashSet:

	public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
	{
		private transient HashMap<E,Object> map;

	    // Dummy value to associate with an Object in the backing Map
	    private static final Object PRESENT = new Object();

	    /**
	     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
	     * default initial capacity (16) and load factor (0.75).
	     */
	    public HashSet() {
	        map = new HashMap<>();
	    }

		//神奇的操作: 默认大小是16，负载因子（load factor)默认值为0.75
	 	public HashSet(Collection<? extends E> c) {
	        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
	        addAll(c);
	    }

		public boolean add(E e) {
       		return map.put(e, PRESENT)==null;
    	}

		...
   	}

  然后会发现一些有趣的事情：

  1. HashSet竟然是用HashMap来实现的。新加入的元素是加入了map中key，value都是同一个Object。

  2. 理解负载因子：  
loadFactor可以看成是容量极限(threshold)/实际容量(capacity)=键值最高对数/实际容量

例如：  
如果负载因子是0.75，hashmap(16)最多可以存储12个元素，想存第16个就得扩容成32。  
如果负载因子是1，hashmap(16)最多可以存储16个元素。

   关于负载因子的问题：

1. 为什么增大负载因子可以减少Hash表所占用的内存空间，但会增加查询数据的时间开销。  
回答：对于存储同样数目的值，负载因子为0.75和1所占的内存空间，是0.75的要多；为1的时候，虽然空间利用率高，但是由于需要解决哈希冲突，需要的额外的开销，增加了查询时间。

2. 为什么减少负载因子可以提高查询数据的性能，但会减低Hash表所占用的内存空间。  
   类似于上面的回答。


站内有一篇介绍哈希表的文章：[Hash 哈希的神奇之处](/algorithm/2015/09/23/java-dataStruct.html)

## 2、 重载与多态无关！
封装可以隐藏实现细节，使得代码模块化；继承可以扩展已存在的代码模块。它们的目的都是为了代码重用。而多态是为了“接口重用”。

Bruce Eckel:"不要犯傻，如果它不是晚绑定，它就不是多态。"

函数的重载不是晚绑定，在编译的时候，重载函数的调用已经是静态的了，它们的地址在编译期就已经确定绑定了，因此，重载和多态无关。

覆盖（Override）和重载（Overload)之间的区别：在派生类重写基类的虚函数，就是重写，重写函数必须有一致的参数表和返回值。  
重载是指写一个已有函数同名但是参数表不同的函数，重载不关心函数的返回值类型。重载不是一种面向对象的编程，而只是一种语法规则。

### 2.1 静态方法不能被非静态方法覆盖

### 2.2 继承的代码复用是做一种“白盒式代码复用”。在继承结构中，父类的内部细节对于之类是可见的。（这在一定程度上破坏了封装性）
组合（composition）是指通过对现有的对象进行封装，产生新的更复杂的功能。因为对象之间，各自的内部实现是不可见的，这属于黑盒式代码复用。
