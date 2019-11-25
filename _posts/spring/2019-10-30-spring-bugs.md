---
layout: post
title: 在Spring中遇到的bug
category: java
comments: false
---

## 1. @Transactional问题

@Transactional会自动提交变更数据，即便函数中没有调用save方法。

@Transactional 的 readOnly 属性默认是false 表示会做数据更新.

假设你底层的Jpa provider用的是Hibernate. Hibernate 在第一级缓存(Session)会自动检测对象的改动,然后触发一个update的调用(导致数据被更新).

如果你不想触发更新改动可以把 @Transactional 的 readOnly 属性设置为true.

This a normal JPA behavior.

Once you retrieve an object via find() or so, that object is regarded as attached, or belongs to a persistence context. Once you exit the method the @Transactional triggers a Spring transaction management aspect which flushes every "dirty" object to database and commits the transaction. Since your object is already changed within the context of the persistence context and the transaction, the changes are saved to the database even without the need to explicitly call a save method.

# REF
> [@Transactional自动提交变更数据](https://www.oschina.net/question/149945_83854?sort=default)
> [Why does @Transactional save automatically to database](https://stackoverflow.com/questions/21552483/why-does-transactional-save-automatically-to-database)