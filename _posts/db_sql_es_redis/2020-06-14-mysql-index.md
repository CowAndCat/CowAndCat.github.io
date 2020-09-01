---
layout: post
title: MySQL索引优化
category: mysql
comments: false
---

MySQL索引优化是考验一个程序员对技术理解和应用的试金石。

# 一、什么时候会用到索引？什么时候索引会失效

## 1.1 什么时候会/不会走索引

全表扫描：（当explain出来的type为ALL时，就是一次全量查询），查询表中的所有数据。

回表：当对一个列创建索引之后，索引会包含该列的键值及键值对应行所在的rowid。通过索引中记录的rowid访问表中的数据就叫回表。回表次数太多会严重影响SQL性能，如果回表次数太多，就不应该走索引扫描，应该直接走全表扫描。

先看看什么情况下不会走索引，或者说索引失效。当索引失效，就只能全表扫描来查询数据。

- 负向条件查询不能使用索引，可以优化为in查询。不等于 （!=/<>）和 is not null ，以及not in/not like/not exists将无法使用到索引
- or在大多数情况下会导致索引失效 （如： select a,b from table where a='a' or b>1 就会导致索引index_a_b失效）  
如果想用or，又走索引， 只能将or条件中的每个列都加上索引：

```
explain SELECT sub_task_id, type FROM plat_sub_task where sub_task_id = 'jfxrnsjgkphxxxyzwvp' or id=2 or create_time='2019-06-22 15:03:35';

type: 'index_merge',
key: 'uk_sub_task_id,PRIMARY,idx_createtime',
key_len: '122,8,4',
Extra: 'Using union(uk_sub_task_id,PRIMARY,idx_createtime); Using where'

```
index merge (Mysql5.0后的技术）说就是在用OR，AND连接的多个查询条件时，可以分别使用前后查询中的索引，然后将它们各自的结果合并交集或并集。 详见：https://dev.mysql.com/doc/refman/5.6/en/index-merge-optimization.html  
注意，index merge要避免deep AND/OR 嵌套SQL，这里有两种优化方式：

```
(x AND y) OR z => (x OR z) AND (y OR z)
(x OR y) AND z => (x AND z) OR (y AND z)
```
另外，Index Merge is not applicable to full-text indexes.

- like 以通配符开头（%aa）会导致索引失效（但是有特殊情况：当查询的列只用到了索引列时，走二级索引查询的数据比一级索引要少，这时“%a”也会用索引）
- 字符串不加单引号，会导致索引失效，因为会存在隐式类型转换。或其他变量类型不匹配的情形（如date和times）
- 数据库执行计算不会命中索引。`EXPLAIN SELECT * FROM user WHERE age+1>24;`就用不到索引
- 单列索引无法储null值，复合索引无法储全为null的值。查询时，采用is null条件时，不能利用到索引，只能全表扫描。（为什么索引列无法存储Null值？索引是有序的。NULL值进入索引时，无法确定其应该放在哪里。）
- 如果MySQL估计使用索引比全表扫描还慢，则不会使用索引。比如小表查询，或大表的大查询
返回数据的比例是重要的指标，比例越低越容易命中索引。记住这个范围值——30%，后面所讲的内容都是建立在返回数据的比例在30%以内的基础上。（https://blog.csdn.net/maihilton/article/details/81564626）

> 官方解释：  
> Indexes are less important for queries on small tables, or big tables where report queries process most or all of the rows. When a query needs to access most of the rows, reading sequentially is faster than working through an index. Sequential reads minimize disk seeks, even if not all the rows are needed for the query.


其他要注意的地方：

- 如果索引了多列，查询从索引的最左前列开始，且不能跳过索引中的列；

假设有个索引 index_a_b_c，那么：
```
select a,b,c from table where a='a' and b>1 and c=4; // 只能用到索引的a,b。 但是否使用到索引与a,b在where出现的顺序无关

```

## 1.2 索引使用分析

有些时候虽然数据库有索引，但是并不被优化器选择使用。

我们可以通过SHOW STATUS LIKE 'Handler_read%';查看索引的使用情况：

Handler_read_key：如果索引正在工作，Handler_read_key的值将很高。  
Handler_read_rnd_next：数据文件中读取下一行的请求数，如果正在进行大量的表扫描，值将较高，则说明索引利用不理想。

用EXPLAIN来分析语句，找准优化方向。

EXPLAIN命令结果中的Using Index意味着不会回表，通过索引就可以获得主要的数据。Using Where则意味着需要回表取数据。
EXPLAIN字段解析：[https://www.runoob.com/w3cnote/mysql-index.html](https://www.runoob.com/w3cnote/mysql-index.html)

实例：
```
mysql> EXPLAIN SELECT `birday` FROM `user` WHERE `birthday` < "1990/2/2"; 
-- 结果： 
id: 1 

select_type: SIMPLE -- 查询类型：简单select（不使用union或子查询）、primary:最外面的select、子查询中的第一个select。。。共8种。

table: user -- 显示这一行的数据是关于哪张表的 。

type: range -- 区间索引（在小于1990/2/2区间的数据)，这是重要的列，显示连接使用了何种类型。从最好到最差的连接类型为system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL。const代表一次就命中，ALL代表扫描了全表才确定结果。一般来说，得保证查询至少达到range级别,最好能达到ref。 

possible_keys: birthday  -- 指出MySQL能使用哪个索引在该表中找到行。如果是空的，没有相关的索引。这时要提高性能，可通过检验WHERE子句，看是否引用某些字段，或者检查字段不是适合索引。  

key: birthday -- 实际使用到的索引。如果为NULL，则没有使用索引。如果为primary的话，表示使用了主键。 

key_len: 4 -- 最长的索引宽度。如果键是NULL，长度就是NULL。在不损失精确性的情况下，长度越短越好。

ref: const -- 显示哪个字段或常数与key一起被使用。  

rows: 1 -- 这个数表示mysql要遍历多少数据才能找到，在innodb上是不准确的。 

Extra: Using where; Using index -- 执行状态说明，这里可以看到的坏的例子是Using temporary和Using filesort
```

SQL 性能优化的目标：至少要达到 range 级别，要求是 ref 级别，如果可以是 consts最好。

其中type(https://blog.csdn.net/dennis211/article/details/78170079)：

- 系统表，少量数据，往往不需要进行磁盘IO；（select * from mysql.time_zone;）
- consts 查找主键索引，返回的数据至多一条（0或者1条）。 属于精确查找（select * from user where id=1;）
- eq_ref 使用了主键或者唯一性索引进行查找，查找结果集只有一个。。属于精确查找.(select * from user,user_ex where user.id=user_ex.id;)
- ref 指的是使用普通的索引（normal index）。意思就是虽然使用了索引，但该索引列的值并不唯一，有重复。所以得多次扫描索引查询重复值。属于精确查找.(select * from user where age=1;)
- range 有范围的索引扫描，相对于index的全索引扫描，它有范围限制，因此要优于index。关于range比较容易理解，需要记住的是出现了range，则一定是基于索引的。同时除了显而易见的between，and以及'>','<'外，in和or也是索引范围扫描。属于范围查找。(select * from user where id between 1 and 4;)
- index, 先走索引物理文件，再全扫描（可能会索引覆盖），速度非常慢。扫描根据索引然后回表取数据，和all相比，他们都是取得了全表的数据，而且index要先读索引而且要回表随机取数据，因此index可能会比all还慢。(select count (*) from user;)
- all 全表扫描，低效且慢

其中Extra:

- 如果是Using index，这意味着信息只用索引树中的信息检索出的，这比扫描整个表要快。
- 如果是Using where used，就是使用上了where限制。
- 如果是impossible where 表示用不着where，一般就是没查出来啥。
- 如果此信息显示Using filesort或者Using temporary的话会很吃力，WHERE和ORDER BY的索引经常无法兼顾，如果按照WHERE来确定索引，那么在ORDER BY时，就必然会引起Using filesort，这就要看是先过滤再排序划算，还是先排序再过滤划算。

## 1.3 using filesort

在使用order by关键字的时候，如果待排序的内容不能由所使用的索引直接完成排序的话，那么mysql有可能就要进行文件排序。（using filesort不一定引起mysql的性能问题）

注意：Using filesort仅仅表示没有使用索引的排序，事实上filesort这个名字很糟糕，并不意味着在硬盘上排序，filesort与文件无关。

filesort是通过相应的排序算法,将取得的数据在内存或硬盘中进行排序。filesort使用的算法是QuickSort，即对需要排序的记录生成元数据进行分块排序，然后再使用mergesort方法合并块。

在MySQL中filesort 的实现算法实际上是有两种：

- 双路排序：是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行指针信息，然后在sort buffer(每个thread独享一片buffer区域）中进行排序。第二次去读取真正要返回的数据的。
- 单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序。

在MySQL4.1版本之前只有第一种排序算法双路排序，第二种算法是从MySQL4.1开始的改进算法，主要目的是为了减少第一次算法中需要两次访问表数据的 IO 操作，将两次变成了一次，但相应也会耗用更多的sortbuffer 空间。当然，MySQL4.1开始的以后所有版本同时也支持第一种算法。

MySQL主要通过比较我们所设定的系统参数 max_length_for_sort_data的大小和Query 语句所取出的字段类型大小总和来判定需要使用哪一种排序算法。如果 max_length_for_sort_data更大，则使用第二种优化后的算法，反之使用第一种算法。所以如果希望 ORDER BY 操作的效率尽可能的高，一定要注意max_length_for_sort_data 参数的设置。如果filesort过程中，由于排序缓存的大小不够大，那么就可能会导致临时表的使用。


优化方向：  
1、修改逻辑，不在mysql中使用order by而是在应用中自己进行排序。  
2、使用mysql索引，将待排序的内容放到索引中，直接利用索引的排序。

例子（https://www.jianshu.com/p/85a4d7b3e5c0）：

（1）explain select id from course where category_id>1 order by category_id;

（2）explain select id from course where category_id>1 order by category_id,buy_times;

根据最左前缀原则，order by后面的的category_id buy_times会用到组合索引，因为索引就是这两个字段

（3）explain select id from course where buy_times > 1 order by category_id;

根据最左前缀原则，order by后面的字段存在于索引中最左列，所以会走索引

（4）explain select id from course where category_id>1 order by buy_times;

（5）explain select id from course where category_id>1 order by buy_times,category_id;

（6）explain select id from course where buy_times > 1 order by buy_times;

根据最左前缀原则，order by后面的字段 没有索引中的最左列的字段，所以不会走索引，会产生using fillesort

（7）explain select id from course order by buy_times desc,category_id asc;

根据最最左前缀原则，order by后面的字段顺序和索引中的不符合，则会产生using filesort

（8）explain select id from course order by category_id desc,buy_times asc;

这一条虽然order by后面的字段和索引中字段顺序相同，但是一个是降序，一个是升序，所以也会产生using filesort，同时升序和同时降序就不会产生using filesort了

## 1.4 using temporary

Using temporary表示由于排序没有走索引、使用union、子查询连接查询、使用某些视图等原因（详见internal-temporary-tables），因此创建了一个内部临时表。注意这里的临时表可能是内存上的临时表，也有可能是硬盘上的临时表，理所当然基于内存的临时表的时间消耗肯定要比基于硬盘的临时表的实际消耗小。

原文链接：https://blog.csdn.net/sz85850597/java/article/details/91907988


## 1.5 索引覆盖

索引覆盖是只查询索引，不走由ID构建的B+Tree。

使用覆盖索引的一个好处是：辅助索引不包含整行记录的所有信息，故其大小要远小于聚集索引，因此可以减少大量的IO操作.
另外一个好处是对某些统计问题而言的, 并不会选择通过查询聚集索引来进行统计。

比如这个例子（也是一个不满足最左前缀原则但是仍旧用了索引的例子）：  
对于（a,b）形式的联合索引，一般是不可以选择b中所谓的查询条件。但如果是统计操作，并且是覆盖索引，则优化器还是会选择使用该索引，如下
```
#联合索引userid_2（userid,buy_date）,一般情况，我们按照buy_date是无法使用该索引的，但特殊情况下：查询语句是统计操作，且是覆盖索引，则按照buy_date当做查询条件时，也可以使用该联合索引
mysql> explain select count(*) from buy_log where buy_date >= '2011-01-01' and buy_date < '2011-02-01';
+--+-----------+-------+-----+-------------+--------+-------+----+----+------------------------+
|id|select_type| table |type |possible_keys| key    |key_len|ref |rows|Extra                   |
+--+-----------+-------+-----+-------------+--------+-------+----+----+------------------------+
| 1| SIMPLE    |buy_log|index| NULL        |userid_2| 8     |NULL|  7 |Using where; Using index|
+--+-----------+-------+-----+-------------+--------+-------+----+----+------------------------+
1 row in set (0.00 sec)
```

索引覆盖还可以用于解决 `limit 20000, 10`的问题。

## 1.6 LIMIT优化
mysql大数据量使用limit分页，随着页码的增大，查询效率越低下。
例如： limit10000,20的意思扫描满足条件的10020行，扔掉前面的10000行，返回最后的20行。

优化方式是 

1. 干掉或者利用 limit offset,size 中的offset
2. 首先获取到offset的id然后直接使用limit size来获取数据。但是要求子查询的列要能在索引中覆盖，不回表。

优化示例：

```
SELECT * FROM xxx WHERE ID > =(select id from table where xxx limit 1000000, 1) limit 20;

SELECT * FROM xxx a JOIN (select id from table where xxx limit 1000000, 20) b ON a.ID = b.id;

SELECT * FROM xxx WHERE ID  BETWEEN (select id from table where xxx limit 1000000, 1) and (select id from table where xxx limit 1000020, 1);
```

# 二、索引的存储原理和工作原理

## 2.1 索引组织结构

B-tree即balance-tree:平衡树:假设1个节点的子节点是5个,平衡树是必须上层节点都满了,才可加到下层.这样树的深度就得到了控制.B-tree除了在叶子节点保存数据,在非叶子节点也保存数据.

B+tree:所有数据都存储在叶子节点,非叶子节点不存储数据.且叶子节点间构成了双向链表。Mysql用的方法是B+tree.

B*tree:也只在叶子节点存储数据并构成双向指针,但在非叶子节点有双向指针。


B+TREE叫聚簇索引(cluster-index).以聚簇索引构建的表叫聚簇索引表。

辅助索引又称二级索引或非聚簇索引.

聚集索引与辅助索引相同的是：不管是聚集索引还是辅助索引，其内部都是B+树的形式，即高度是平衡的，叶子结点存放着所有的数据。

聚集索引与辅助索引不同的是：叶子结点存放的是否是一整行的信息。

辅助索引的存在并不影响数据在聚集索引中的组织，因此每张表上可以有多个辅助索引，但只能有一个聚集索引。

当通过辅助索引来寻找数据时，InnoDB存储引擎会遍历辅助索引并通过叶子级别的指针获得只想主键索引的主键，然后再通过主键索引来找到一个完整的行记录。

## 2.2 联合索引

从本质上来说，联合索引就是一棵B+树，不同的是联合索引的键值得数量不是1，而是>=2。

在第一个键相同的情况下，也已经对第二个键进行了排序处理。

## 2.3 索引下推
ICP: Index Condition Pushdown(索引下推)

ICP技术是在MySQL5.6中引入的一种索引优化技术。它能减少在使用二级索引过滤where条件时的回表次数和减少MySQL server层和引擎层的交互次数。在索引组织表中，使用二级索引进行回表的代价相比堆表中是要高一些的。

在了解ICP（Index Condition Pushdown，索引条件下推）特性之前，必须先搞明白**where查询条件的提取规则**：所有SQL的where条件，均可归纳为3大类：Index Key (First Key & Last Key)，Index Filter，Table Filter。 参见：https://www.cnblogs.com/Terry-Wu/p/9273177.html

1.1 Index First Key

只是用来定位索引的起始范围，因此只在索引第一次Search Path(沿着索引B+树的根节点一直遍历，到索引正确的叶节点位置)时使用，一次判断即可；

1.2 Index Last Key

用来定位索引的终止范围，因此对于起始范围之后读到的每一条索引记录，均需要判断是否已经超过了Index Last Key的范围，若超过，则当前查询结束；

2. Index Filter

用于过滤索引查询范围中不满足查询条件的记录，因此对于索引范围中的每一条记录，均需要与Index Filter进行对比，若不满足Index Filter则直接丢弃，继续读取索引下一条记录；

3. Table Filter

则是最后一道where条件的防线，用于过滤通过前面索引的层层考验的记录，此时的记录已经满足了Index First Key与Index Last Key构成的范围，并且满足Index Filter的条件，回表读取了完整的记录，判断完整记录是否满足Table Filter中的查询条件，同样的，若不满足，跳过当前记录，继续读取索引的下一条记录，若满足，则返回记录，此记录满足了where的所有条件，可以返回给前端用户。

对Using index condition的理解是，首先mysql server和storage engine是两个组件，server负责sql的parse执行； storage engine去真正的做数据/index的读取/写入。

在MySQL 5.6以前是这样：server命令storage engine按index key把相应的数据从数据表读出，传给server，然后server来按where条件（index filter和table filter）做选择。  
而在MySQL 5.6加入ICP后，Index Filter与Table Filter分离，Index Filter下降到InnoDB的索引层面进行过滤，如果不符合条件则无须读数据表，减少了回表与返回MySQL Server层的记录交互开销，节省了disk IO，提高了SQL的执行效率。

a 当关闭ICP时,index 仅仅是data access 的一种访问方式，存储引擎通过索引回表获取的数据会传递到MySQL Server层进行where条件过滤。

b 当打开ICP时,如果部分where条件能使用索引中的字段,MySQL Server 会把这部分下推到引擎层,可以利用index过滤的where条件在存储引擎层进行数据过滤,而非将所有通过index access的结果传递到MySQL server层进行where过滤.

优化效果:ICP能减少引擎层访问基表的次数和MySQL Server 访问存储引擎的次数；减少磁盘io次数，提高查询语句性能.


# 三、索引的优化方向

## 3.1 设计
索引设计的原则

- 适合索引的列是出现在where子句中的列，或者连接子句中指定的列；

- 基数较小的类，索引效果较差，没有必要在此列建立索引；（区分度不大的字段上不宜建立索引）

- 使用短索引，如果对长字符串列进行索引，应该指定一个前缀长度，这样能够节省大量索引空间；

- 不要过度索引。索引需要额外的磁盘空间，并降低写操作的性能。在修改表内容的时候，索引会进行更新甚至重构，索引列越多，这个时间就会越长。所以只保持需要的索引有利于查询即可。

- 更新十分频繁的字段上不宜建立索引：因为更新操作会变更B+树，重建索引。这个过程是十分消耗数据库性能的。

- 不要抵制唯一索引。业务上具有唯一特性的字段，即使是多个字段的组合，也必须建成唯一索引。虽然唯一索引会影响insert速度（先查后插），但是对于查询的速度提升是非常明显的。另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，在并发的情况下，依然有脏数据产生。


建组合索引的时候，区分度最高的在最左边。  
正例：如果 where a=? and b=?，a 列的几乎接近于唯一值，那么只需要单建 idx_a 索引即可。  
说明：存在非等号和等号混合判断条件时，在建索引时，请把等号条件的列前置。如：where c>? and d=?
那么即使 c 的区分度更高，也必须把 d 放在索引的最前列，即建立组合索引 idx_d_c。

## 3.2 范围查询
如果是范围查询和等值查询同时存在，优先匹配等值查询列的索引：

EXPLAIN SELECT * FROM user WHERE status>5 AND age=24;

我的理解是，如果同时存在 index_age_xx 和 index_status, 会优先匹配index_age_xx索引。

## 3.3 in 和 exists
外层查询表小于子查询表，则用exists，外层查询表大于子查询表，则用in，如果外层和子查询表差不多，则爱用哪个用哪个。

select * from A where id EXISTS (select b.id from B); // B>A  
select * from A where id IN (select b.id from B); // B<A

## 3.4 当in的数据量很大的时候

1. 建议将IN改为INNER JOIN查询，并尽量控制传入参数的数量，分批次小数据量地查询，以保证不会因为单条SQL影响数据库整体性能。
2. 仍使用in子查询，多查询一次（利用索引覆盖）
```
SELECT * FROM basic_zdjbxx WHERE suiji IN ( SELECT zdcode FROM basic_h WHERE zdcode != "" )
优化后：
SELECT * FROM basic_zdjbxx WHERE suiji IN ( SELECT zdcode FROM ( SELECT zdcode FROM basic_h WHERE zdcode != "" ) AS h )
```
这篇文章提到了in查询的原理：https://www.cnblogs.com/xh831213/archive/2012/05/09/2491272.html

    1）关于使用IN的子查询：
    Consider the following statement that uses an uncorrelated subquery:

    SELECT ... FROM t1 WHERE t1.a IN (SELECT b FROM t2);
    The optimizer rewrites the statement to a correlated subquery:

    SELECT ... FROM t1 WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.b = t1.a);

    If the inner and outer queries return M and N rows, respectively, the execution time becomes on the order of O(M×N), rather than O(M+N) as it would be for an uncorrelated subquery.

    An implication is that an IN subquery can be much slower than a query written using an IN(value_list) construct that lists the same values that the subquery would return.

    2）关于把子查询转换成join的：
    The optimizer is more mature for joins than for subqueries, so in many cases a statement that uses a subquery can be executed more efficiently if you rewrite it as a join.

    An exception occurs(下面是个反例) for the case where an IN subquery can be rewritten as a SELECT DISTINCT join. Example:

    SELECT col FROM t1 WHERE id_col IN (SELECT id_col2 FROM t2 WHERE condition);
    That statement can be rewritten as follows:

    SELECT DISTINCT col FROM t1, t2 WHERE t1.id_col = t2.id_col AND condition;
    But in this case, the join requires an extra DISTINCT operation and is not more efficient than the subquery

## 3.5 not in的优化方式
http://www.piaoyi.org/database/MYSQL-not-in-left-join.html

采用left join 和 右表.id is null 的方法优化

https://blog.csdn.net/hongsejiaozhu/article/details/1876181
连接（JOIN）.. 之所以更有效率一些，是因为 MySQL不需要在内存中创建临时表来完成这个逻辑上的需要两个步骤的查询工作 。

# REF
> https://dev.mysql.com/doc/refman/5.6/en/optimize-overview.html
