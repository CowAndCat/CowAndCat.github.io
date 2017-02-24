---
layout: post
title: SQL笔试题3
category: sql
comments: false
---
## 1、PreparedStatement与Statement比较
- PreparedStatement可以写动态参数化的查询，如：

		SELECT interest_rate FROM loan WHERE loan_type=?

	占位符的索引位置从1开始而不是0。

- PreparedStatement比 Statement 更快：前者拥有更佳的性能优势，SQL语句会预编译在数据库系统中。执行计划同样会被缓存起来，它允许数据库做参数化查询。使用预处理语句比普通的查询更快，因为它做的工作更少（**数据库对SQL语句的分析，编译，优化已经在第一次查询前完成了**）。

	为了减少数据库的负载，生产环境中的JDBC代码你应该总是使用PreparedStatement 。  

	值得注意的一点是：为了获得性能上的优势，应该使用参数化sql查询而不是字符串追加的方式。否则就不是正确使用PreparedStatement的查询。

- PreparedStatement可以防止SQL注入式攻击：在使用参数化查询的情况下，数据库系统（eg:MySQL）不会将参数的内容视为SQL指令的一部分来处理，**而是在数据库完成SQL指令的编译后，才套用参数运行**，因此就算参数中含有破坏性的指令，也不会被数据库所运行。

- 比起凌乱的字符串追加似的查询，PreparedStatement查询可读性更好、更安全。

### PreparedStatement的局限性

尽管PreparedStatement非常实用，但是它仍有一定的限制。

1. 为了防止SQL注入攻击，PreparedStatement不允许一个占位符（？）有多个值，在执行有**IN**子句查询的时候这个问题变得棘手起来。下面这个SQL查询使用PreparedStatement就不会返回任何结果

		SELECT * FROM loan WHERE loan_type IN (?)
		preparedSatement.setString(1, "'personal loan', 'home loan', 'gold loan'");

2. PreparedStatement的第一次执行消耗是很高的。它的性能体现在后面的重复执行。
