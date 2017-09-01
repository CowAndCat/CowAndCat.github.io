---
layout: post
title: Spring framework源码导读（1）
category: spring
comments: false
---

## 1. 俯瞰Spring源码架构

### 1.1 系统架构

Spring 总共大约有20个模块，由1300多个不同的文件构成。而这些组件被分别整合在核心容器（Core Container）、Aop（Aspect Oriented Programming）和设备支持（Instrmentation）、数据访问及集成（Data Access/Integeration）、Web、报文发送（Messaging）、Test，6个模块集合中。

以下是 Spring 4 的系统架构图。

![images](/images/201706/springframework.png)
 
组成 Spring 框架的每个模块集合或者模块都可以单独存在，也可以一个或多个模块联合实现。每个模块的组成和功能详见下文。

### 1.2 核心容器

由spring-beans、spring-core、spring-context和spring-expression（Spring Expression Language, SpEL） 4个模块组成。

core包侧重于帮助类，操作工具，beans包更侧重于bean实例的描述。context更侧重全局控制，功能衍生。

spring-beans和spring-core模块是Spring框架的核心模块，包含了控制反转（Inversion of Control, IoC）和依赖注入（Dependency Injection, DI）。BeanFactory 接口是Spring框架中的核心接口，它是工厂模式的具体实现。BeanFactory 使用控制反转对应用程序的配置和依赖性规范与实际的应用程序代码进行了分离。但 BeanFactory 容器实例化后并不会自动实例化 Bean，只有当 Bean 被使用时 BeanFactory 容器才会对该 Bean 进行实例化与依赖关系的装配。

spring-context模块构架于核心模块之上，他扩展了BeanFactory，为她添加了Bean生命周期控制、框架事件体系以及资源加载透明化等功能。此外该模块还提供了许多企业级支持，如邮件访问、远程访问、任务调度等，ApplicationContext是该模块的核心接口，她是 BeanFactory 的超类，与 BeanFactory 不同，ApplicationContext 容器实例化后会自动对所有的单实例 Bean 进行实例化与依赖关系的装配，使之处于待用状态。 

spring-expression模块是统一表达式语言（unified EL）的扩展模块，可以查询、管理运行中的对象，同时也方便的可以调用对象方法、操作数组、集合等。它的语法类似于传统EL，但提供了额外的功能，最出色的要数函数调用和简单字符串的模板函数。这种语言的特性是基于 Spring 产品的需求而设计，他可以非常方便地同Spring IoC进行交互。

包结构：
![core](/images/core.jpg)

### 1.3 Aop和设备支持

由spring-aop、spring-aspects和spring-instrumentation 3个模块组成。

spring-aop是Spring的另一个核心模块，是Aop主要的实现模块。作为继OOP后，对程序员影响最大的编程思想之一，Aop极大地开拓了人们对于编程的思路。在Spring中，他是以JVM的动态代理技术为基础，然后设计出了一系列的Aop横切实现，比如前置通知、返回通知、异常通知等，同时，Pointcut接口来匹配切入点，可以使用现有的切入点来设计横切面，也可以扩展相关方法根据需求进行切入。

spring-aspects模块集成自AspectJ框架，主要是为Spring Aop提供多种Aop实现方法。

 spring-instrumentation模块是基于java SE中的"java.lang.instrument"进行设计的，应该算是Aop的一个支援模块，主要作用是在JVM启用时，生成一个代理类，程序员通过代理类在运行时修改类的字节，从而改变一个类的功能，实现Aop的功能。在分类里，我把他分在了Aop模块下，在Spring 官方文档里对这个地方也有点含糊不清，这里是纯个人观点。

### 1.4 数据访问及集成

由spring-jdbc、spring-tx、spring-orm、spring-jms和spring-oxm 5个模块组成。

spring-jdbc模块是Spring 提供的JDBC抽象框架的主要实现模块，用于简化Spring JDBC。主要是提供JDBC模板方式、关系数据库对象化方式、SimpleJdbc方式、事务管理来简化JDBC编程，主要实现类是JdbcTemplate、SimpleJdbcTemplate以及NamedParameterJdbcTemplate。

spring-tx模块是Spring JDBC事务控制实现模块。使用Spring框架，它对事务做了很好的封装，通过它的Aop配置，可以灵活的配置在任何一层；但是在很多的需求和应用，直接使用JDBC事务控制还是有其优势的。其实，事务是以业务逻辑为基础的；一个完整的业务应该对应业务层里的一个方法；如果业务操作失败，则整个事务回滚；所以，事务控制是绝对应该放在业务层的；但是，持久层的设计则应该遵循一个很重要的原则：保证操作的原子性，即持久层里的每个方法都应该是不可以分割的。所以，在使用Spring JDBC事务控制时，应该注意其特殊性。

spring-orm模块是ORM框架支持模块，主要集成 hibernate, Java Persistence API (JPA) 和 Java Data Objects (JDO) 用于资源管理、数据访问对象(DAO)的实现和事务策略。

spring-jms模块（Java Messaging Service）能够发送和接受信息，自Spring Framework 4.1以后，他还提供了对spring-messaging模块的支撑。

spring-oxm模块主要提供一个抽象层以支撑OXM（OXM是Object-to-XML-Mapping的缩写，它是一个O/M-mapper，将java对象映射成XML数据，或者将XML数据映射成java对象），例如：JAXB, Castor, XMLBeans, JiBX 和 XStream等。

### 1.5 Web

由spring-web、spring-webmvc、spring-websocket和spring-webmvc-portlet 4个模块组成。

spring-web模块为Spring提供了最基础Web支持，主要建立于核心容器之上，通过Servlet或者Listeners来初始化IoC容器，也包含一些与Web相关的支持。

spring-webmvc模块众所周知是一个的Web-Servlet模块，实现了Spring MVC（model-view-controller）的Web应用。

spring-websocket模块主要是与Web前端的全双工通讯的协议。（资料缺乏，这是个人理解）

spring-webmvc-portlet模块是知名的Web-Portlets模块（Portlets在Web门户上管理和显示的可插拔的用户界面组件。Portlet产生可以聚合到门户页面中的标记语言代码的片段，如HTML，XML等），主要是为SpringMVC提供Portlets组件支持。

### 1.6 报文发送

即spring-messaging模块。

spring-messaging是Spring4 新加入的一个模块，主要职责是为Spring 框架集成一些基础的报文传送应用。

### 1.7 Test

即spring-test模块。

spring-test模块主要为测试提供支持的，毕竟在不需要发布（程序）到你的应用服务器或者连接到其他企业设施的情况下能够执行一些集成测试或者其他测试对于任何企业都是非常重要的。


