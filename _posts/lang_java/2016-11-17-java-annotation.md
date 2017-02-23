---
layout: post
title: Java Annotation
category: java
comments: false
---
面试 Baidu的时候被面到了，So master it.

## 1. 注解的优劣势

## 2. 构成

### 2.1 元注解

元注解的作用就是负责注解其他注解。

Java5.0定义了4个标准的meta-annotation类型，它们被用来提供对其它 annotation类型作说明。

Java5.0定义的元注解：

	1. @Target
    2. @Retention
    3. @Documented
    4. @Inherited

这些类型和它们所支持的类在java.lang.annotation包中可以找到。下面一一介绍。

#### 2.1.1 @Target

@Target说明了Annotation所修饰的对象范围：Annotation可被用于 packages、types（类、接口、枚举、Annotation类型）、类型成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环变量、catch参数）。在Annotation类型的声明中使用了target可更加明晰其修饰的目标。

作用：用于描述注解的使用范围（即：被描述的注解可以用在什么地方）

取值(ElementType)有：

	1. CONSTRUCTOR:用于描述构造器
	2. FIELD:用于描述域
	3. LOCAL_VARIABLE:用于描述局部变量
	4. METHOD:用于描述方法
	5. PACKAGE:用于描述包
	6. PARAMETER:用于描述参数
	7. TYPE:用于描述类、接口(包括注解类型) 或enum声明

实例：

	@Target(ElementType.TYPE)
	public @interface Table {
	    /**
	     * 数据表名称注解，默认值为类名称
	     * @return
	     */
	    public String tableName() default "className";
	}

	@Target(ElementType.FIELD)
	public @interface NoDBColumn {

	}

注解Table 可以用于注解类、接口(包括注解类型) 或enum声明,而注解NoDBColumn仅可用于注解类的成员变量。

#### 2.1.2 @Retention

@Retention定义了该Annotation被保留的时间长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的Annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取（请注意并不影响class的执行，因为Annotation与class在使用上是被分离的）。

使用这个meta-Annotation可以对 Annotation的“生命周期”限制。

作用：表示需要在什么级别保存该注释信息，用于描述注解的生命周期（即：被描述的注解在什么范围内有效）

取值（RetentionPoicy）有：

	1. SOURCE:在源文件中有效（即源文件保留）
	2. CLASS:在class文件中有效（即class保留）
	3. RUNTIME:在运行时有效（即运行时保留）

#### 2.1.3 @Documented
@Documented用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化。

Documented是一个标记注解，没有成员。


#### 2.1.4 @Inherited
@Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

注意：@Inherited annotation类型是被标注过的class的子类所继承。类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation。

当@Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层。


## 3. 自定义注解
使用@interface自定义注解时，自动继承了java.lang.annotation.Annotation接口，由编译程序自动完成其他细节。##在定义注解时，不能继承其他的注解或接口。## @interface用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是基本类型、Class、String、enum）。可以通过default来声明参数的默认值。


## 参考
> [深入理解Java：注解（Annotation）自定义注解入门](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)