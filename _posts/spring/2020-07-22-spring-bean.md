---
layout: post
title: 在Spring中bean的生命周期
category: spring
comments: false
---

# Spring如何管理Bean的生命周期

Spring IOC 容器对 bean 的生命周期进行管理的过程：

- 通过构造器或者工厂方法创建 bean 实例。
- 为 bean 的属性赋值和对其他 bean 的引用。
- 调用 bean 的初始化方法。
- bean 初始成功，可以使用。
- 容器关闭时 , 调用 bean 的销毁方法。

在 bean 的声明里设置 init-method 和 destroy-method 属性 , 为 bean 指定初始化和销毁方法。

```
   <bean id="initBean" class="com.itdjx.spring.beans.init.InitBean" init-method="init" destroy-method="destroy">
       <property name="name" value="func"/>
       <property name="info" value="my name is ET"/>
   </bean>
```

init-method和InitializingBean接口方法afterPropertiesSet的先后顺序是：

**先调用InitializingBean的afterPropertiesSet方法，再调用init-method方法。**


# @Value, @Autowired, @PostConstruct的执行顺序
@PostConstruct注解好多人以为是Spring提供的。其实是Java自己的注解.

Java中该注解的说明：@PostConstruct该注解被用来修饰一个非静态的void（）方法。被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。PostConstruct在构造函数之后执行，init（）方法之前执行。

通常我们会是在Spring框架中使用到@PostConstruct注解 该注解的方法在整个Bean初始化中的执行顺序：

Constructor(构造方法) -> @Autowired(依赖注入) -> @Value -> @PostConstruct(注释的方法)


# REF 
> [Spring学习-- IOC 容器中 bean 的生命周期](https://www.cnblogs.com/chinda/p/6491490.html)