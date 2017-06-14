---
layout: post
title: junit mock
category: java
comments: false
---

## 1. 为什么要使用mock？

mock是一类工具的简称，最大的功能是把单元测试的耦合分解开，当依赖链很长的时候能发挥奇效。

## 2. 工具汇总

可用的Mock Toolkit有许多，比较常见的有EasyMock, Jmock，mockito和JMockit等等。

### 2.1 EasyMock (比较早，常用）

[EasyMock 使用方法与原理剖析](https://www.ibm.com/developerworks/cn/opensource/os-cn-easymock/)

EasyMock 是一套用于通过简单的方法对于给定的接口生成 Mock 对象的类库。  

它提供对接口的模拟，能够通过录制、回放、检查三步来完成大体的测试过程， 
可以验证方法的调用种类、次数、顺序，可以令 Mock 对象返回指定的值或抛出指定异常。

单测过程大致可以划分为以下几个步骤： 

- 使用 EasyMock 生成 Mock 对象；
- 设定 Mock 对象的预期行为和输出；
- 将 Mock 对象切换到 Replay 状态；
- 调用 Mock 对象方法进行单元测试；
- 对 Mock 对象的行为进行验证。

### 2.2 jMock

[http://www.jmock.org/](http://www.jmock.org/)

### 2.3 mockito

Java mocking is dominated by expect-run-verify libraries like EasyMock or jMock. Mockito offers simpler and more intuitive approach: you ask questions about interactions after execution. Using mockito, you can verify what you want. Using expect-run-verify libraries you are often forced to look after irrelevant interactions.
 
No expect-run-verify also means that Mockito mocks are often ready without expensive setup upfront. They aim to be transparent and let the developer to focus on testing selected behavior rather than absorb attention.

### 2.4 jmockit

[http://jmockit.org/](http://jmockit.org/)

### 2.5  PowerMock

[https://github.com/powermock/powermock](https://github.com/powermock/powermock)

### 2.6 Unitils 

Unitils构建在DBUnit与EasyMock项目之上并与JUnit和TestNG相结合。支持数据库测试，支持利用mock对象进行测试并提供与spring和hibernate相集成。Unitils设计成以一种高度可配置和松散偶合的方式来添加这些服务到单元测试中.