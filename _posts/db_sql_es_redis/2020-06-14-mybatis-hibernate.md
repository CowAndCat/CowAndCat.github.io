---
layout: post
title: Mybatis VS hibrnate
category: sql
comments: false
---

# 一、Hibernate与Mybatis对比
## 1.1 缓存机制对比

Hibernate一级缓存是Session缓存，利用好一级缓存就需要对Session的生命周期进行管理好。建议在一个Action操作中使用一个Session。一级缓存需要对Session进行严格管理。

Hibernate二级缓存是SessionFactory级的缓存。 SessionFactory的缓存分为内置缓存和外置缓存。内置缓存中存放的是SessionFactory对象的一些集合属性包含的数据(映射元素据及预定SQL语句等),对于应用程序来说,它是只读的。外置缓存中存放的是数据库数据的副本,其作用和一级缓存类似.二级缓存除了以内存作为存储介质外,还可以选用硬盘等外部存储设备。二级缓存称为进程级缓存或SessionFactory级缓存，它可以被所有session共享，它的生命周期伴随着SessionFactory的生命周期存在和消亡。

MyBatis: 默认情况下是没有开启缓存的,除了局部的 session 缓存,可以增强变现而且处理循环 依赖也是必须的。要开启二级缓存,你需要在你的 SQL 映射文件中添加一行:  `<cache/>`

相同点: Hibernate和Mybatis的二级缓存除了采用系统默认的缓存机制外，都可以通过实现你自己的缓存或为其他第三方缓存方案，创建适配器来完全覆盖缓存行为。

不同点: 

Hibernate的二级缓存配置在SessionFactory生成的配置文件中进行详细配置，然后再在具体的表-对象映射中配置是那种缓存。

MyBatis的二级缓存配置都是在每个具体的表-对象映射中进行详细配置，这样针对不同的表可以自定义不同的缓存机制。并且Mybatis可以在命名空间中共享相同的缓存配置和实例，通过Cache-ref来实现。

两者比较

因为Hibernate对查询对象有着良好的管理机制，用户无需关心SQL。所以在使用二级缓存时如果出现脏数据，系统会报出错误并提示。而MyBatis在这一方面，使用二级缓存时需要特别小心。如果不能完全确定数据更新操作的波及范围，避免Cache的盲目使用。否则，脏数据的出现会给系统的正常运行带来很大的隐患。

## 1.2 Hibernate优势和缺点

Hibernate的DAO层开发比MyBatis简单，Mybatis需要维护SQL和结果映射。

Hibernate对对象的维护和缓存要比MyBatis好，对增删改查的对象的维护要方便。

Hibernate数据库移植性很好，MyBatis的数据库移植性不好，不同的数据库需要写不同SQL。

Hibernate有更好的二级缓存机制，可以使用第三方缓存。MyBatis本身提供的缓存机制不佳。

总结优势：Hibernate功能强大，数据库无关性好，O/R映射能力强

Hibernate的缺点就是学习门槛不低，要精通门槛更高，而且怎么设计O/R映射，在性能和对象模型之间如何权衡取得平衡，以及怎样用好Hibernate方面需要你的经验和能力都很强才行。 

## 1.3 Mybatis优势和缺点

MyBatis可以进行更为细致的SQL优化，可以减少查询字段。

MyBatis容易掌握，而Hibernate门槛较高。

缺点就是框架还是比较简陋，功能尚有缺失（二级缓存机制不佳），虽然简化了数据绑定代码，但是整个底层数据库查询实际还是要自己写的，工作量也比较大，而且不太容易适应快速数据库修改。


# 二、mybatis一级缓存二级缓存
https://www.cnblogs.com/happyflyingpig/p/7739749.html

一级缓存：

Mybatis对缓存提供支持，但是在没有配置的默认情况下，它只开启一级缓存，一级缓存只是相对于同一个SqlSession而言。所以在参数和SQL完全一样的情况下，我们使用同一个SqlSession对象调用一个Mapper方法，往往只执行一次SQL，因为使用SelSession第一次查询后，MyBatis会将其放在缓存中，以后再查询的时候，如果没有声明需要刷新，并且缓存没有超时的情况下，SqlSession都会取出当前缓存的数据，而不会再次发送SQL到数据库。

怎么判断某两次查询是完全相同的查询？

mybatis认为，对于两次查询，如果以下条件都完全一样，那么就认为它们是完全相同的两次查询。

- 传入的statementId相同
- 查询时要求的结果在前一次的结果范围内
-  这次查询所产生的最终要传递给JDBC java.sql.Preparedstatement的Sql语句字符串（boundSql.getSql() ）
- 传递给java.sql.Statement要设置的参数值

SqlSession的一级缓存性能问题（https://blog.csdn.net/chenyao1994/article/details/79233725）：

1. MyBatis对会话（Session）级别的一级缓存设计的比较简单，就简单地使用了HashMap来维护，并没有对HashMap的容量和大小进行限制。
2. 一级缓存是一个粗粒度的缓存，没有更新缓存和缓存过期的概念

二级缓存：

MyBatis的二级缓存是Application级别的缓存，它可以提高对数据库查询的效率，以提高应用的性能。

# 三、MyBatis 中 ${}和 #{}千万不要乱用

`#{}`是预编译处理，MyBatis在处理`#{ }`时，它会将sql中的`#{ }`替换为？，然后调用PreparedStatement的set方法来赋值，传入字符串后，会在值两边加上单引号，如上面的值 “4,44,514”就会变成“ ‘4,44,514’ ”；

`${}`是字符串替换，在处理时是字符串替换，MyBatis在处理`${}`时,它会将sql中的`${}`替换为变量的值，传入的数据不会加两边加上单引号。

使用${ }会导致sql注入，不利于系统的安全性！SQL注入：就是通过把SQL命令插入到Web表单提交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。常见的有匿名登录（在登录框输入恶意的字符串）、借助异常获取数据库信息等

# REF 
> https://www.cnblogs.com/williamjie/p/9198987.html  
> https://www.cnblogs.com/jiangxiaoyaoblog/p/5633075.html