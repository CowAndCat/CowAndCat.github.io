---
layout: post
title: Spring framework源码-核心部分
category: spring
comments: false
---
# 1. spring-core 阅读
- 阅读之前，应温习关于Java reflect/annotation的知识。
- 先看org.springframework.lang里面的标注，这些标注会贯穿整片Spring代码，先搞懂没有坏处。

    - @Nonnull 加上这个标注的对象不能为Null；
    - @NonNullApi 一个加了@Nonnull标签的标签，用在method和parameter上 
    - @Nullable The annotated element could be null under some circumstances.

- 在大部分package里，会有一个package-info.java 文件来介绍包的信息。里面没有类，可能就一行代码：package org.springframework.cglib; 里面的重要信息都在注释里。
- asm spring独立的asm程序？和汇编有关系？
- cglib - Byte Code Generation Library is high level API to generate and transform Java byte code. It is used by AOP, testing, data access frameworks to generate dynamic proxy objects and intercept field access. （API，用于生成java bytecode）

- 主要的代码集中在core里。annotation包是一些解析或构建标注的类；codec包里是一些de/encode 资源、字节流的工具类；convert包里是一些做类型转换的API（在converter里）和一套实现（support,支持基础类型的相互转换）；env包里是一些与环境配置相关的类；io包是关于文件和资源的输入输出类；serializer是关于序列化的；style是为了在转String时输出多种格式而存在的类；task是一些关于调度的东西。

> #### 科普时刻-UUID   
> 
> UUID含义是通用唯一识别码 (Universally Unique Identifier)，这是一个软件建构的标准，也是被开源软件基金会 (Open Software Foundation, OSF) 的组织应用在分布式计算环境 (Distributed Computing Environment, DCE) 领域的重要部分。  
> 
> UUID是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的。计算用到了以太网卡地址、纳秒级时间、芯片ID码和许多可能的数字.  
> 
> UUID的唯一缺陷在于生成的结果串会比较长。(128-bits)
 ------
> #### 科普时刻-Lambda
> 
> Java 中的 Lambda 表达式通常使用 (argument) -> {body} 语法书写。
> 
> 参数的类型既可以明确声明，也可以根据上下文来推断。
> 
> 当只有一个参数或body只有一行语句的时候，括号或花括号可以省略。
> 
> 例如：
> 以下是一些 Lambda 表达式的例子：
> 
>        (int a, int b) -> {  return a + b; }
>        () -> System.out.println("Hello World");
>        (String s) -> { System.out.println(s); }
------
> #### 科普时刻-@see 和 {@link}
>  写JavaDoc的时候，用 @see 和 {@link}能够关联类或方法。  
>  比如：
>  ```@see fully-qualified-classname#方法名称/属性名词```  
>  它们的区别在于：
>  
>      + @see 只能单独一行顶头写，如果不顶头写就不管用了，没了链接的效果
>      + @link 可以随便放

 
- 遇到一个很有意思的接口 IntPredicate。  
是一个接受单个int值、返回boolean值的predicate。可以这样用：

        import java.util.function.IntPredicate; 
        public class Main {
          public static void main(String[] args) {
            IntPredicate i = (x)-> x < 0;        
            System.out.println(i.test(123));
            // Output: false
          }
        }

- 代码量还是很巨大的，姑且走马观花地看。
- 突然间Get到的一个阅源小方法：如果源码看不懂，可以结合unit test来看。
- 然而UT并不能直接运行，因为spring-core还依赖cglib和objenesis，所以我在build.gradle文件里的project("spring-core")->dependencies里增加了两行：

        compile group: 'cglib', name: 'cglib', version: '3.2.5'
        compile group: 'org.objenesis', name: 'objenesis', version: '2.6'

配置重新import后，到cglib和objenesis里更新一下package import，然后编译就能通过，ut也能运行了。
(其实这里有一个repackaging技术，就是不将cglib的代码耦合到spring中，而是在用项目的时候，将cglib的代码重新打包到对应的spring的cglib里面。)

# 2. spring-beans 阅读

- Spring里将bean作为操作单元，这个子项目包含的内容是对bean进行操作的接口和类。比如，init beans, 注入内容，检查是否为singleton等等。
- Autowired/Qualifier/Required等标签的定义就是在这个模块里。
- 大部分内容是对properties进行编辑操作的，集中在propertyeditors这个包里。
- 例如 ClassEditor，借助thread context ClassLoader来操作类中属性，能将一个字符串转换成类对象的属性。一个应用可以参考：[java.beans.PropertyEditor(属性编辑器)简单应用](http://www.blogjava.net/orangewhy/archive/2007/06/26/126371.html)

> #### 科普时刻-Prototype Pattern
> 原型模式是一种创建型设计模式,它通过复制一个已经存在的实例来返回新的实例,而不是新建实例.被复制的实例就是我们所称的原型,这个原型是可定制的.  
> 原型模式多用于**创建复杂的或者耗时的实例**, 因为这种情况下,复制一个已经存在的实例可以使程序运行更高效,或者创建值相等,只是命名不一样的同类数据.

# 3. spring-context 阅读

(There is) Spring's central application context runtime. Also includes scheduling and remoting abstractions.

-  笔者认为schedule部分很有学习的价值。
    
        @Nullable
        ScheduledFuture<?> schedule(Runnable task, Trigger trigger);
        // 当触发器生效，调度任务task

- There is no necessity for sth./sb. to do sth.
