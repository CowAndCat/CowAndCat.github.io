---
layout: post
title: mysql的分页优化
category: mysql
comments: false
---

## 一、分页优化

    select * from table1 limit m, n

随着偏移量 m 的增大，MySQL 需要花费大量时间来扫描需要丢弃的数据。

优化方法：**延迟关联，通过使用覆盖索引查询返回需要的主键**，再根据这些主键和原表做一次关联操作获得需要的行，这可以减少回表次数，同时减少 MySQL 扫描那些需要丢弃的行数。

比如id上有索引，优化方法有（一种是id>=的形式，另一种就是利用join）：

    select * from table1 where id >= (select id from table1 limit m, 1) limit 10

    select * from table1 a join (select id from table1 limit m, 10) b on a.id = b.id

    select * from table1 inner join (select id from table1 limit m, 10) as x using id

执行子查询时，MySQL 需要创建临时表，查询完毕后再删除这些临时表，所以，子查询的速度会比关联查询低。

其他优化语句：（这里设置了索引key(k, id)）

    select * from table1 a, 
    (select id from table1 where k > 1000 limit m, 10) b 
    where a.id = b.id

    select * from table1 a, 
    (select id from table1 where k > 1000 and id > 100000 order by id limit 10) b 
    where a.id = b.id

    select * from (
        select id from table1
        where k > 1000 order by id limit 300000,10
    ) a left join table1 b on a.id = b.id

