---
layout: post
title: SQL笔试题1
category: sql
comments: false
---
## 1. 标准SQL创立索引：

	create [unique] [cluster] index <索引名> on <表名> （<列名> [<次序>], ...);

如：

	create unique index s_id on student(id dsc);

## 2. COALESCE 函数（ORACLE）：
	返回表达式(用,分割）中第一个非null的值。

## 3. 获取mysql中的版本和当前时间：select version(), now();

## 4. REGEXP:支持正则表达式

	select * from pet where name REGEXP '^b';

表示选择所有name字段中以b开头的记录。

## 5. 选择前多少条数据：利用top，group by,以及DESC

	select TOP 3 name form student ORDER BY grade DESC；

## 6. SQL Server数据库中删除表格中完全相同的数据（考察distinct）

先将不同的记录copy到一个新的表格，删除原来表格中的记录，再将没有重复的数据copy回去。

	select distinct * into tmpTable from table;
	delete from table;
	insert into table select * from tmpTable;
	drop table tmpTable;

## 7. 使用JDBC的步骤
1. 注册数据库驱动
2. 创建Connection类型对象
3. 获得Statement类型对象
4. 执行查询或者更新
5. 处理ResultSet类型对象
6. 释放资源

## 8. MySQL能通过MySQLDataSource直接建立数据源

## 9. Connection接口提供了哪些事务控制方法？
1. setAutoCommit()
2. commit()
3. rollback()

## 10. 查询每门课程成绩都大于80分学生的学号

数据库 表- student

	name score course
	A 	85  语文
	A 	75  数学
	A 	82  英语
	B 	75  语文
	B 	89  数学
	B 	79  英语

每门课程都超过80分的那个人名字的SQL语句:

SQL1:

	select name from test.stu
	group by name
	having count(score) =sum(case  when score>80 then 1 else 0 end )

SQL2:

	select name from stu
	group by name
	having name not in (
	select name from stu
	where score <80)

SQL3:

	select name from test.stu
	group by name
	having min(score)>=80


## 11. 查询课程001的成绩大于课程002成绩的学号

student表：sno（学号），sname（姓名），sex（性别），dept(系）  
course课程表：cno（课程号），课程名（cname）   
sc选课表：sno，cno，grade（成绩）

	select cno from sc a inner join (select * from sc where cno=(select cno from course where cname='001')) as b on a.cno>o=(select cno from course where cname='002')

## 12、选出depart表（did, name)和employee(eid,name, depart_id)中的如下数据：

	did name employee_sum  
	1	财务		2  
	2	销售		0  
	3	客服		2  

SQL:  

	select depart.*, ifnull(num, 0) num from depart
	left join( select did, count(*) num from employee group by did) e
	on e.did=depart.did;

考点：ifnull/left join/ count 和 group by的组合

更多参考：[sql语句](http://blog.sina.com.cn/s/blog_8ea826d10102vm1h.html)

## 13、连接
内连接是保证两个表中所有的行都要满足连接条件，而外连接则不然。
在外连接中，某些不满条件的列也会显示出来，也就是说，只限制其中一个表的行，而不限制另一个表的行。

- JOIN (INNER JOIN): 如果表中有至少一个匹配，则返回行. 例如：from table1, table2 (‘，’就可以实现连接）
- （下面全是outer join)LEFT JOIN: 即使右表中没有匹配，也从左表返回所有的行。left join是left outer join的简写。
- RIGHT JOIN: 即使左表中没有匹配，也从右表返回所有的行
- FULL JOIN: 只要其中一个表中存在匹配，就返回行union 操作符用于合并两个或多个 SELECT 语句的结果集。请注意，UNION 内部的 SELECT 语句必须拥有相同数量的列。列也必须拥有相似的数据类型。同时，每条 SELECT 语句中的列的顺序必须相同。
- union all 会选取所有的列，相等的会出现两次

## REF
> [MySQL面试题集锦，据说知名互联网公司都用](http://tech.it168.com/a2017/1119/3180/000003180421.shtml)
