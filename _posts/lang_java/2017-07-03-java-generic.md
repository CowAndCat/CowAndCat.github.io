---
layout: post
title: Java 泛型
category: java
comments: false
---

泛型的一个最主要的作用是减少重复代码。

泛型最主要的优点就是让编译器追踪参数类型，执行类型检查和类型转换：编译器保证类型转换不会失败。

要理解思想；另外要为解决实际问题而用，而不是为了用泛型而用。

# 1. 基础概念

泛型类，是在实例化类的时候指明泛型的具体类型；  
泛型方法，是在调用方法的时候指明泛型的具体类型。

泛型类语法：

    public class TestClassDefine<T, S extends T>{}

泛型方法语法格式：

![img](/images/201707/generic.png)

定义泛型方法时，必须在返回值前边加一个<T>，来声明这是一个泛型方法，持有一个泛型T，然后才可以用泛型T作为方法的返回值。

Class<T>的作用就是指明泛型的具体类型，而Class<T>类型的变量c，可以用来创建泛型类的对象。

当赋值的类型不确定的时候，用通配符(?)代替。

为什么要用变量c来创建对象呢？  
既然是泛型方法，就代表着我们不知道具体的类型是什么，也不知道构造方法如何，因此没有办法去new一个对象，但可以利用变量c的newInstance方法去创建对象，也就是利用反射创建对象。

# 2.  一些区别

在Java泛型中，？ 表示通配符，代表未知类型，< ? extends Object>表示上边界限定通配符，< ? super Object>表示下边界限定通配符。

### 2.1 通配符 与 T 的区别

T：作用于模板上，用于将数据类型进行参数化，不能用于实例化对象。   
?：在实例化对象的时候，不确定泛型参数的具体类型时，可以使用通配符进行对象定义。

    < T > 等同于 < T extends Object>
    < ? > 等同于 < ? extends Object>

例一：定义泛型类，将key，value的数据类型进行< K, V >参数化，而不可以使用通配符。

    public class Container<K, V> {
        private K key;
        private V value;

        public Container(K k, V v) {
            key = k;
            value = v;
        }
    }

例二：实例化泛型对象，我们不能够确定eList存储的数据类型是Integer还是Long，因此我们使用List<? extends Number>定义变量的类型。

    List<? extends Number> eList = null;
    eList = new ArrayList<Integer>();
    eList = new ArrayList<Long>();

### 2.2 上界类型通配符（? extends）与 下界类型通配符（? super ）

上界类型通配符例子：

    List<? extends Number> eList = null;
    eList = new ArrayList<Integer>();
    Number numObject = eList.get(0);  //语句1，正确

    //Type mismatch: cannot convert from capture#3-of ? extends Number to Integer
    Integer intObject = eList.get(0);  //语句2，错误

    //The method add(capture#3-of ? extends Number) in the type List<capture#3-of ? extends Number> 
    // is not applicable for the arguments (Integer)
    eList.add(new Integer(1));  //语句3，错误

语句1：List<? extends Number>eList存放Number及其子类的对象，语句1取出Number（或者Number子类）对象直接赋值给Number类型的变量是符合java规范的。  

语句2：List<? extends Number>eList存放Number及其子类的对象，语句2取出Number（或者Number子类）对象直接赋值给Integer类型（Number子类）的变量是不符合java规范的。 

语句3：List<? extends Number>eList不能够确定实例化对象的具体类型，因此无法add具体对象至列表中，可能的实例化对象如下。

    eList = new ArrayList<Integer>();
    eList = new ArrayList<Long>();
    eList = new ArrayList<Float>();

总结：上界类型通配符add方法受限，但可以获取列表中的各种类型的数据，并赋值给父类型（extends Number）的引用。因此如果你想从一个数据类型里获取数据，使用 ? extends 通配符。限定通配符总是包括自己。

下界类型通配符（? super ）例子：

    List<? super Integer> sList = null;
    sList = new ArrayList<Number>();

    //Type mismatch: cannot convert from capture#5-of ? super Integer to Number
    Number numObj = sList.get(0);  //语句1，错误

    //Type mismatch: cannot convert from capture#6-of ? super Integer to Integer
    Integer intObj = sList.get(0);  //语句2，错误

    sList.add(new Integer(1));  //语句3，正确

语句1：List<? super Integer> 无法确定sList中存放的对象的具体类型，因此sList.get获取的值存在不确定性，子类对象的引用无法赋值给兄弟类的引用，父类对象的引用无法赋值给子类的引用，因此语句错误。 

语句2：同语句1。 

语句3：子类对象的引用可以赋值给父类对象的引用，因此语句正确。 

总结：下界类型通配符get方法受限，但可以往列表中添加各种数据类型的对象。因此如果你想把对象写入一个数据结构里，使用 ? super 通配符。限定通配符总是包括自己。

### 2.3 注意事项

- 如果你想从一个数据类型里获取数据，使用 ? extends 通配符
- 如果你想把对象写入一个数据结构里，使用 ? super 通配符
- 如果你既想存，又想取，那就别用通配符
- 不能同时声明泛型通配符上界和下界
- 静态变量不能够使用泛型定义。（public static T value; //错误的定义 ）
- 泛型的定义不会被继承，举个例子来说，如果A是B的子类，而C是一个声明了泛型定义的类型的话，C&lt;A>不是C&lt;B>的子类。（List&lt;Number> 并不是 List&lt;Integer> 的父类）


# 3. 原生类型的autoboxing和auto-unboxing (装箱和拆箱)

常见的语句：

    int i=10;
    Integer b=i;//自动的装箱
    int k=b;//自动的拆箱

我们知道，在Java中，int,long等原生类型不是一个继承自Object的类，所以相应的，有很多操作我们都不能利用原生类型操作，比如想要把一个整数放入到一个集合中，我们必须首先创建一个Integer对象，然后再将这个对象放入到集合中。  

当我们从集合中取数的时候，取出来的是一个Integer对象，因此不能直接对它使用加减乘除等运算符，而是必须用Integer.intValue()取到相应的值才可以，这样的过程称之为boxing和unboxing。

autoboxing和auto-unboxing能为我们省掉了很多不必要的工作，但是会额外消耗性能。

    long t = System.currentTimeMillis();
    Long sum = 0L;
    for (long i = 0; i < Integer.MAX_VALUE; i++) {
        sum += i;
    }
    System.out.println("total:" + sum);
    System.out.println("processing time: " + (System.currentTimeMillis() - t) + " ms");

输出：

    total:2305843005992468481
    processing time: 9803 ms

将 Long sum = 0L; 换成 long sum = 0L;

输出：

    total:2305843005992468481
    processing time: 1343 ms





