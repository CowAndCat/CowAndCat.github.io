---
layout: post
title: Mockito编写单元测试的一个陷阱
category: java
comments: false
---

在使用Mockito的when thenReturn来mock数据的时候，发现when中的函数仍旧会被执行。

经过排查，发现是因为when中的函数添加了final修饰符，这是Mockito的一个坑。

实际上，将这个坑归咎于Mockito是欠妥当的，因为可以通引入依赖来解决。

添加 mockito-inline jar依赖即可。
https://mvnrepository.com/artifact/org.mockito/mockito-inline

或者换成PowerMock。
