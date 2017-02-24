---
layout: post
title: having, group by and where
category: sql
comments: false
---

## 一、 having和where的区别
having子句与where有相似之处但也有区别,都是设定条件的语句。

在查询过程中聚合语句(sum,min,max,avg,count)要比having子句优先执行。而where子句在查询过程中执行优先级别优先于聚合语句(sum,min,max,avg,count)。

简单说来：

- where子句：

		select sum(num) as rmb from order where id>10 //只有先查询出id大于10的记录才能进行聚合语句

-	having子句:

		select reportsto as manager, count(*) as reports from employees
		group by reportsto having count(*) > 4

以northwind库为例。  
having条件表达式为聚合语句。肯定的说having子句查询过程执行优先级别低于聚合语句。  

再换句说话说把上面的having换成where则会出错。统计分组数据时用到聚合语句。  

对分组数据再次判断时要用having。如果不用这些关系就不存在使用having。直接使用where就行了。

**having就是来弥补where在分组数据判断时的不足。因为where执行优先级别要快于聚合语句。**

## 二、聚合函数

聚合函数，这是必需先讲的一种特殊的函数：
例如SUM, COUNT, MAX, AVG等。这些函数和其它函数的根本区别就是它们一般作用在多条记录上。

	SELECT SUM(population) FROM tablename

这里的SUM作用在所有返回记录的population字段上，结果就是该查询只返回一个结果，即所有国家的总人口数。   

通过使用GROUP BY 子句，可以让SUM 和 COUNT 这些函数对属于一组的数据起作用。  

当你指定 GROUP BY region 时， 属于同一个region（地区）的一组数据将只能返回一行值.

也就是说，表中所有除region（地区）外的字段，只能通过 SUM, COUNT等聚合函数运算后返回一个值．
HAVING子句可以让我们筛选成组后的各组数据．

**HAVING子句在聚合后对组记录进行筛选，而WHERE子句在聚合前先筛选记录．也就是说作用在GROUP BY 子句和HAVING子句前。**

## 三、几个例子

一、显示每个地区的总人口数和总面积．

	SELECT region, SUM(population), SUM(area)   
	FROM bbc  
	GROUP BY region

先以region把返回记录分成多个组，这就是GROUP BY的字面含义。分完组后，然后用聚合函数对每组中的不同字段（一或多条记录）作运算。

二、 显示每个地区的总人口数和总面积．仅显示那些面积超过1000000的地区。  

	SELECT region, SUM(population), SUM(area)  
	FROM bbc  
	GROUP BY region  
	HAVING SUM(area)>1000000  

在这里，我们不能用where来筛选超过1000000的地区，因为表中不存在这样一条记录。

相反，HAVING子句可以让我们筛选成组后的各组数据．

三、

以下示例使用的数据库是MySQL 5。  

	数据表：student  
	表结构：  
	Field Name DataType Len  
	id                int           20  
	name           varchar    25  
	major           varchar    25  
	score           int           20  
	sex              varchar    20  

表数据：

	编号/姓名/专业/学分/性别  
	id   name major     score sex  
	1    jak    Chinese    40    f  
	2    rain    Math        89    m  
	3    leo    Phy          78    f  
	4    jak    Math         76    f  
	5    rain    Chinese   56    m  
	6    leo    Math         97    f  
	7    jak    Phy          45    f  
	8    jak    Draw         87    f  
	9    leo    Chinese    45    f  

现在我们要得到一个视图：  

- 要求查询性别为男生，并且列出每个学生的总成绩：  

		SQL：  
		select s.*,sum(s.score) from student s where sex='f' group by s.name

		Result:
		id   name major     score sex sum(s.score)  
		1    jak    Chinese    40    f       248  
		3    leo    Phy         78     f       220  

	可以看到总共查到有两组，两组的学生分别是jak和leo，每一组都是同一个学生，这样我们就可以使用聚合函数了。  
	只有使用了group by语句，才能使用如：count()、sum()之类的聚合函数。

- 下面我们再对上面的结果做进一步的筛选，只显示总分数大于230的学生：  

		SQL：  
		select s.*,sum(s.score) from student s where sex='f' group by s.name having sum(s.score)>230

		Result:
		id   name major     score       sex   sum(s.score)  
		1    jak    Chinese    40          f       248

**可见having与where的功能差不多。**

## 四、结论
1. WHERE 子句用来筛选 FROM 子句中指定的操作所产生的行。  
2. GROUP BY 子句用来分组 WHERE 子句的输出。  
3. HAVING 子句用来从分组的结果中筛选行。  

