---
layout: post
title: Java 基础知识(gc,JVM等）
category: java
comments: false
---
阅读《高质量Java程序设计》（顾晓刚等编著 2003）、
《Java程序设计语言（第四版）》（[美]ken arnold等，陈佛爷等译，2006）（这本书很值得一看）

### 1、ClassLoader

程序员可以调用ClassLoader.defineClass()方法来动态加载类。处于安全考虑，ClassLoader禁止程序员手工载入任何来自于Java类库中的类，否则你就有机会加载一个可以执行恶意代码的非官方的String类！这是defineClass方法调用约定的一部分。

Java中强化这个预定的代码：

	protected final Class defineClass(String...) throws ClassFormatError{
		check();
		if(name != null && name.startsWith("java.")){
			throw new SecurityException("Prohibited package name:"+name.subString(0, name.lastIndexOf('.')));
	}

Q: 如果自定义实现一个String类，所有的属性和方法都和jdk里面的一样，如果使用到String，会使用哪个？

看包名，如果不是java.lang.String,会用自定义的String。

当一个装载器被请求装载某个类时，它首先委托自己的parent去装载，若parent能装载，则返回这个类所对应的Class对象，若parent不能装载，则由parent的请求者去装载。
当前装载器的parent会去装载com.test.String类,结果没有找到.则由当前装载器去装载

如果包名改为java.lang.String
当前装载器的parent会去装载java API 中的java.lang.String类,并返回这个类所对应的Class对象
所以自定义的java.lang.String就不会被装载。

### 2、代码风格
- 良好的注释
- 尽量不使用import*
- 控制代码的长度，但不能过量，如嵌套的三元表达式。
- 尽量减少同名的类

### 3、内存管理
C++的析构函数，一个重要的功能就是释放内存，而Java的finalize函数则完全不是为了释放内存而设计的。

Java中的内存泄露和C++中的内存泄露也有本质的区别。与其称为内存泄露，不如称为对内存的误用。

可达但是已经不会再使用的对象是内存泄露的源泉。避免方法， 是尽量少使用长期的引用。

通过profiling工具，可以发现内存泄露，并诊断出发生内存泄露的对象。

#### 3.1 Java中的内存管理
内存分配有三种：  

- 从静态存储区域分配。内存在编译的时候就分配好了，如static变量；
- 在栈上创建；
- 在堆中创建。用new来创建对象。

堆上的空间由自动的存储管理系统进行控制，即由gc进行控制。

#### 3.2 JVM内存模型
除了栈和堆，JVM中海油其他一些内存区域，例如方法区（method area）、常量池（Constant pool）、本地栈（native method area）等。

### 4、gc

#### 4.1 分代复制垃圾收集器  
算法基于这样一个假设：超过95%的对象的生命周期都很短。

  该算法根据对象的生命周期分为两代，所有新创建的对象都在一个类似栈的内存区域进行分配，这个区域叫eden。当这块内存已经全部分配给对象（区域满了），其中大多数对象已经“死亡”，然后收集器把未死亡的对象复制到另一块内存中，直接更新eden的指针就行了。  

  生存在eden中对象称为年轻代，长期对象叫年老代。

  该方法效率很高,因为清空区域花费的时间极少（只要更新指针），当old generation的内存区域全部分配完时，收集器会进行一次主要垃圾收集，这次就会比较慢。

#### 4.2 标记垃圾收集器
在上面提到的年老代中，用标记垃圾收集器回收。

先从一组根引用开始，遍历所有的对象，如果对象被根引用，标记“存活”，存活对象的引用也同样标记“存活”，如此循环，其余未被标记的即认为“死亡”，将被回收。

根据对存活对象的处理方式，可以分为两种：

**1**.标记紧缩垃圾收集器（mark-and-compact collector）：将所有存活对象复制到一个连续的内存区域中，减少垃圾碎片。

**2**.标记清除垃圾收集器（mark-and-sweep collector):将死亡对象的内存空间记录到一个自由的空间列表中。后续的新来的old generation按空间表插入到原来死亡对象位置。会产生较多的内存碎片。

上面两个收集器在运行时，会停止JVM中其他程序的线程，而且会收集所有的垃圾内存，因此停顿时间不可预测！
（其实分代也会操作停顿，但是相对时间较短，仍是一个问题）

#### 4.3 分代收集   
现在的虚拟机垃圾收集大多采用这种方式，它根据对象的生存周期，将堆分为新生代和老年代。在新生代中，由于对象生存期短，每次回收都会有大量对象死去，那么这时就采用 复制 算法。老年代里的对象存活率较高，没有额外的空间进行分配担保，所以可以使用 标记 - 整理（1）   或者   标记 - 清除 （2）

#### 4.4 增量垃圾收集
能够提供接近于常数的固定的暂停时间，用户感觉不到这个时间。

原理：将时间较长的年老代的垃圾回收分成许多较短的间隔完成，每次回收一部分。

-Xms 指定堆大小的下限  
-Xmx 指定堆大小的上限  
如果相等，堆的大小不发生变化。

### 5、Java程序性能调优
1. 不要使用new String("string");方法来创建String对象。
2. 使用StringBuffer代替迭代使用的String，因为前者是可变的，操作直接在引用对象上。
3. 用Double.parseDouble(doubleStr)代替Double.valueOf(doubleStr). 这样能少创建对象，该规则同样适合于Integer、Short和Long。（不适合JDK1.2，因为低版本没有parseDouble)


### 6、Java中内存回收（手动）
Java虽然保证自动回收所有的未被引用的对象，但现在的java虚拟机并不会检测对象的最后一次引用在什么地方！

如果要手工清除短期对象的引用，最有效的方法是使对象指向null。

当然，还有有效的方法：引入Reference类型。（weak类型在内存不足时可以自动地被回收）
如:

	Vector v = new Vector();
	WeakReference wr = new WeakReference(v);
	v=null;
	System.gc(); //敦促gc运行
	if(((Vector) wr.get())==null){
		System.out.println("Weak referenced object collected");
	}


### 7、StringBuilder & StringBuffer

StringBuffer类基本上等同于StringBuilder类，它们之间只存在一个差异：StringBuffer类为可追加的字符序列提供了线程安全的实现。

次要差异：buffer比较旧一点。

### 8、？通配符
Java中的？是一个通配符（wildcard），来指定类型参数。读作“未指定类型的”或“某类型的”。

有一种方法可以限制通配符所代表的类型：

	void processValues(String[]names, Lookup<? extends Number> table){
		for(int i=0; i<names.length; i++){
			Number value=table.find(names[i]);
			if(value!=null)
				processValue(names[i], value);

		}
	}

上面`? extends Number`表示类型Number以及任何扩展或实现Number的类型。

### 9、import static method;
Java中是可以引入静态方法的，使用和引入包差不多。如引入Math.abs()方法：
` import static java.lang.Math.abs;`
后续使用只要直接调用abs即可。

但是很容易造成命名冲突，一条准则是：为了提高代码的可读性和净化代码。

### 10、本地方法（native method），JNI，native关键字
使用：
	public native int getCPUID();

当想使用某些现有的非Java编程语言编写的代码，或者我们需要直接操作某种硬件，那么可以编写本地方法。

JNI是指Java Native Interface，使得java能调用本地方法。

JNI一个重要用处是提高Java程序的性能，另外，也可以用来实现与平台相关的功能。（两个优点）

JNI的缺点：使用了JNI，就会失去大部分java平台的好处，例如自动垃圾回收、平台无关性等。

### 11、继承中，类中元素的访问控制
- 类中的静态成员（无论是字段Field还是方法Method）是不可以被覆盖的，只能被隐藏。调用隐藏方法的办法是super.staticMethod()。

### 12、instanceof
- 该操作符用来检测一个对象所属的类。
- 当instanceof作用于null时总是返回false！

### 13、protected
一个受保护的成员可以被类本身以及与其在同一个包中的代码访问，还可以在一个类中通过对象引用来访问。（想到组合模式）

### 14、Interface
接口中可以声明具名常量，但是它们都隐式地是public、static和final的。

所有的方法隐式都是abstract的。

与类不同的是，接口可以扩展多个接口：
`public interface SerializableRunnable extends java.io.Serializable, Runnable`

### 15、不常见的关键字
- strictfp
	- strictfp 关键字可应用于类、接口或方法。
	- 使用 strictfp 关键字声明一个方法时，该方法中所有的float和double表达式都严格遵守FP-strict的限制,符合IEEE-754规范。
	- 当对一个类或接口使用 strictfp 关键字时，该类中的所有代码，包括嵌套类型中的初始设定值和代码，都将严格地进行计算。
	- 如果你想让你的浮点运算更加精确，而且不会因为不同的硬件平台所执行的结果不一致的话，可以用关键字strictfp.
	- 如： `public strictfp class MyClass {  }`

- transient
	- 当串行化某个对象时，如果该对象的某个变量是 transient，那么这个变量不会被串行化进去。也就是说，假设某个类的成员变量是transient，那么当通过ObjectOutputStream把这个类的某个实例保存到磁盘上时，实际上transient变量的值是不会保存的。因为当从磁盘中读出这个对象的时候，对象的该变量会没有被赋值。
	- 如：`private transient String pwd;`

- volatile
	- 用volatile修饰的变量，线程在每次使用变量的时候，都会读取变量修改后的最新的值。
	- 对于volatile修饰的变量，jvm虚拟机只是保证从主内存加载到线程工作内存的值是最新的，所以，volatile很容易被误用，用来进行原子性操作，其实还是会存在并发的情况。
	- `public volatile static int count = 0;`

### 16、join
join方法用来等待另一个线程终止，最简单的形式是一直等待某个特定线程结束。


### 17、实现interface和继承class的区别？
多继承的难处：一是无法处理多个父类之间域的冲突；而是难于确定调用的是哪个父类的方法。

interface实际上提供了一种简化的处理方法， 首先interface中只能包含常量，这使得域冲突降低了；其次interface中只包含方法的定义，于是不同接口之间的方法也不会冲突。

### 20、clone和serialization有什么区别，有什么共同之处？
它们都可以看做是一种复制对象的方法，这是共同之处。

clone默认的是浅层拷贝，按位复制对象，也可以实现clone机制来完成深层拷贝。clone总是发生在同一个JVM中。

serialization则是深层拷贝，会复制一张完整的“对象网”，对象会被写入内存之外的存储器中，并在不同的jvm之间传递。

### 21、Java创建对象的几种方式（重要）
(1) 用new语句创建对象，这是最常见的创建对象的方法。  

(2) 运用反射手段,调用java.lang.Class或者java.lang.reflect.Constructor类的newInstance()实例方法。

(3) 调用对象的clone()方法。

(4) 运用反序列化手段，调用java.io.ObjectInputStream对象的 readObject()方法。

(1)和(2)都会明确的显式的调用构造函数 ；(3)是在内存上对已有对象的影印，所以不会调用构造函数 ；(4)是从文件中还原类的对象，也不会调用构造函数。

### 22、ArrayList list = new ArrayList(20);中的list扩充几次（）A 0     B 1     C 2      D 3

答案：A解析：这里有点迷惑人，大家都知道默认ArrayList的长度是10个，所以如果你要往list里添加20个元素肯定要扩充一次（扩充为原来的1.5倍），但是这里显示指明了需要多少空间，所以就一次性为你分配这么多空间，也就是不需要扩充了。

### 23、下面程序能正常运行吗（）
	public class NULL {
	    public static void haha(){
	        System.out.println("haha");
	    }
	    public static void main(String[] args) {
	        ((NULL)null).haha();
	    }
	}
答案：能正常运行解析：输出为haha。

因为null值可以强制转换为任何java类类型,(String)null也是合法的。

但null强制转换后是无效对象，其返回值还是为null，而static方法的调用是和类名绑定的，不借助对象进行访问所以能正确输出。反过来，没有static修饰就只能用对象进行访问，使用null调用对象肯定会报空指针错了。这里和C++很类似。
