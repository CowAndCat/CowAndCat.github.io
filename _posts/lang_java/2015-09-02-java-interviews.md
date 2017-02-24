---
layout: post
title: Java面试题
category: java
comments: false
---
## 1. Java实现多态的机制  
父类与子类之间的多态方式：方法的重写。  
覆盖父类的方法，是动态绑定的，在运行时确定调用的是哪个方法 。  
类内部的多态方式：方法的重载。

## 2. 自动拆包和装包是？  
即自动完成基本数据类型与对应的包装类型的转换。


## 3. char占16个位，即2个byte，能存放一个汉字。
char取值是0到2的16次方减1。short也就16位，是有符号的。

## 4. 创建字符串的方式。  
3种：直接常量赋值；String构造函数；方法实例化valueOf.

## 5. null和""，==和equals()的区别  
null和“”:前面没有分配内存，后面分配了。
==和equals:前者比较的是内存地址和内容;后者比较的是内容，不考虑地址。

## 6.  switch 条件表达式类型: int, short, char，byte。(要是低于int位数的数据类型）

## 7.  为什么要有面向对象？

* 容易理解，能够和需求联系起来。（扩展性）
* 易于维护，软件可读性，可修改性和可测试性得到增强(灵活性)
* 利于重用。

## 8. Arrays是一个工具类，包含用来操作数组（比如排序和搜索）的各种方法。  
* 对于基础类型，Arrays类中的sort()使用的是“经过调优的快速排序法”;对于Object数组排序则是使用的是合并排序。
* 比如int[]，double[]，char[]等基数据类型的数组，Arrays类之只是提供了默认的升序排列，没有提供相应的降序排列方法。

## 9. Array和Arrays两个类什么区别？Collection和Collections什么区别？
* Array类主要提供了动态创建和访问 Java 数组的方法。Arrays包含用来操作数组（比如排序和搜索）的各种方法。此类还包含一个允许将数组作为列表来查看的静态工厂。
* Collections是个java.util下的类，它包含有各种有关集合操作的静态方法。他提供一系列静态方法实现对各种集合的搜索、排序、线程安全化等操作。   Collection是个java.util下的接口，它是各种集合结构的父接口  继承与他的接口主要有Set 和List.

## 10. 优先级队列（PriorityQueue）是不同于先进先出队列的另一种队列。每次从队列中取出的是具有最高优先权的元素。

* 如果不提供Comparator的话，优先队列中元素默认按自然顺序排列，也就是数字默认是小的在队列头，字符串则按字典序排列。
* 优先队列不允许空值，而且不支持non-comparable（不可比较）的对象，比如用户自定义的类。
* 优先队列的大小是不受限制的，但在创建时可以指定初始大小。当我们向优先队列增加元素的时候，队列大小会自动增加。
* PriorityQueue是非线程安全的，所以Java提供了PriorityBlockingQueue（实现BlockingQueue接口）用于Java多线程环境。

## 11. map中没有iterator,有的是entry。iterator仅用于集合类。

## 12. 下面代码的输出结果是什么，解释一波。
	public class Fun {
		public static void main(String[] args) {
			new Son();
		}
	}

	class Father {
		private static String i = "father static";
		static {
			System.out.println(i);
		}

		public Father() {
			System.out.println("father constructor");
		}

		private int j = func();

		public int func() {
			System.out.println("father non static");
			return 1;
		}
	}

	class Son extends Father {
		private static String i = "son static";
		static {
			System.out.println(i);
		}

		public Son() {
			System.out.println("son constructor");
		}

		private int j = func();

		public int func() {
			System.out.println("son non static");
			return 1;
		}
	}

输出结果：

	father static  
	son static  
	son non static  
	father constructor  
	son non static  
	son constructor  

你需要通过这段程序，知道Java继承中，static、成员初始化和构造函数的调用顺序。

## 13.Java中wait和notify有什么作用？并用它们实现一个BlockingQueue，要求能通过构造函数定义队列空间，有put操作和take操作，分别是添加元素和删除元素。

- wait作用：调用某个对象的wait()方法**能让当前线程阻塞**，并且当前线程必须拥有此对象的monitor（即锁）。

- notify()方法：**能够唤醒一个正在等待这个对象的monitor的线程**，如果有多个线程都在等待这个对象的monitor（锁），则只能唤醒其中一个线程；

- notifyAll()方法：能够唤醒所有正在等待这个对象的monitor的线程；

	注意这三个函数都是在object上的：由于每个对象都拥有monitor（即锁），所以让当前线程等待某个对象的锁，当然应该通过这个对象来操作了。而不是用当前线程来操作，因为当前线程可能会等待多个线程的锁，如果通过线程来操作，就非常复杂了。

实现（类似于ArrayBlockingQueue)：

	import java.util.ArrayList;
	import java.util.List;

	public class BlockingQueue<T> {

		private int size;
		private int capacity;
		private List<T> cache;

		public BlockingQueue(int capcacity) {
			this.capacity = capcacity;
			cache = new ArrayList<T>(capcacity);
			this.size = 0;
		}

		public synchronized void take() throws InterruptedException {
			this.notify();
			while (size <= 0) {
				this.wait();
			}
			cache.remove(0);
			size--;
		}

		public synchronized void put(T t) throws InterruptedException {
			this.notify();
			while (size >= capacity) {
				this.wait();
			}
			cache.add(t);
			size++;
		}
	}

### 需要注意的地方：

1. 先唤醒其他线程，再放弃自己的锁，即先notify()/notifyAll()，再wait()
2.  wait() 方法必须放在while循环中，不能放在if 条件中
3.  一般情况下，notifyAll()  优于 notify()。

利用生成者和消费者问题进行测试：

	public class Test {
		public static void testProduceAndConsume() {
			// 水果篮子
			final Basket basket = new Basket();

			// 生产者线程
			class Producer implements Runnable {
				public void run() {
					while (true) {
						try {
							System.out.println("Produce apples begin ");
							basket.produce();
							System.out.println("Produce apples end ");
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					}
				}
			}

			// 消费者线程
			class Consumer implements Runnable {
				public void run() {
					while (true) {
						try {
							System.out.println("Comsume apples begin ");
							basket.consume();
							System.out.println("Comsume apples end");
						} catch (InterruptedException e) {
							e.printStackTrace();
						}
					}
				}
			}

			Producer producer = new Producer();
			Consumer consumer = new Consumer();

			// 开始两个进程
			Thread t1 = new Thread(producer);
			Thread t2 = new Thread(consumer);

			t1.start();
			t2.start();
		}

		public static class Basket {
			// 篮子只能放2个apple
			BlockingQueue<Integer> basket = new BlockingQueue<Integer>(2);

			public void produce() throws InterruptedException {
				basket.put(1);
			}

			public void consume() throws InterruptedException {
				basket.take();
			}
		}

		public static void main(String[] args) {
			testProduceAndConsume();
		}
	}

将BlockingQueue用Java内置的ArrayBlockingQueue替换，能得到同样的效果。

## 14.BlockingQueue

BlockingQueue定义的常用方法如下：

- add(anObject)：
    把anObject加到BlockingQueue里，如果BlockingQueue可以容纳，则返回true，否则抛出异常。

- offer(anObject)：
    表示如果可能的话，将anObject加到BlockingQueue里，即如果BlockingQueue可以容纳，则返回true，否则返回false。

- put(anObject)：
     把anObject加到BlockingQueue里，如果BlockingQueue没有空间，则调用此方法的线程被阻断直到BlockingQueue里有空间再继续。

- poll(time)：
    取走BlockingQueue里排在首位的对象，若不能立即取出，则可以等time参数规定的时间，取不到时返回null。

- take()：
     取走BlockingQueue里排在首位的对象，若BlockingQueue为空，阻断进入等待状态直到BlockingQueue有新的对象被加入为止。

BlockingQueue有四个具体的实现类，根据不同需求，选择不同的实现类：

- ArrayBlockingQueue：
    规定大小的BlockingQueue，其构造函数必须带一个int参数来指明其大小。其所含的对象是以FIFO（先入先出）顺序排序的。

- LinkedBlockingQueue：
     大小不定的BlockingQueue，若其构造函数带一个规定大小的参数，生成的BlockingQueue有大小限制，若不带大小参数，所生成的BlockingQueue的大小由Integer.MAX_VALUE来决定。其所含的对象是以FIFO顺序排序的。

- PriorityBlockingQueue：
     类似于LinkedBlockingQueue,但其所含对象的排序不是FIFO，而是依据对象的自然排序顺序或者是构造函数所带的
Comparator决定的顺序。

- SynchronousQueue：
   特殊的BlockingQueue，对其的操作必须是放和取交替完成的。

LinkedBlockingQueue和ArrayBlockingQueue比较起来，它们背后所用的数据结构不一样，导致LinkedBlockingQueue的数据吞吐量要大于ArrayBlockingQueue，但在线程数量很大时其性能的可预见性低于ArrayBlockingQueue。

## 15. for循环可以不使用{}，但是仅限于执行语句（不包括变量声明语句！）

	for (int i = 0; i < 10; i++){
		int k=i;
	}
	//不加花括号，竟然会编译出错
	for (int i = 0; i < 10; i++)
		int k=i;
