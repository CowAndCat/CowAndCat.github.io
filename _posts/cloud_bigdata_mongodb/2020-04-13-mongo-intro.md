---
layout: post
title: MongoDB简要介绍
category: mongodb
comments: false
---
# 1. 介绍
MongoDB 是一个基于分布式文件存储的数据库。由 C++ 语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。

MongoDB 是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。

## 1.1 和RDBMS的对应关系
下表列出了 RDBMS 与 MongoDB 对应的术语：

|RDBMS   |MongoDB
|--|--
|数据库 |数据库
|表格  |集合
|行   |文档
|列   |字段
|表联合 |嵌入文档
|主键  |主键 (MongoDB 提供了 key 为 _id )

mongodb的document是BSON格式，松散的.

## 1.2 ObjectId

ObjectId 类似唯一主键，可以很快的去生成和排序，包含 12 bytes，含义是：

- 前 4 个字节表示创建 unix 时间戳,格林尼治时间 UTC 时间，比北京时间晚了 8 个小时
- 接下来的 3 个字节是机器标识码
- 紧接的两个字节由进程 id 组成 PID
- 最后三个字节是随机数

MongoDB 中存储的文档必须有一个 \_id 键。这个键的值可以是任何类型的，默认是个 ObjectId 对象

由于 ObjectId 中保存了创建的时间戳，所以你不需要为你的文档保存时间戳字段，你可以通过 getTimestamp 函数来获取文档的创建时间:
```
> var newObject = ObjectId()
> newObject.getTimestamp()
ISODate("2019-11-25T07:21:10Z")
```

## 1.3 基本操作
参考： [https://www.runoob.com/mongodb/mongodb-tutorial.html](https://www.runoob.com/mongodb/mongodb-tutorial.html)

MongoDB 使用 update() 和 save() 方法来更新集合中的文档:

- update() 方法用于更新已存在的文档。如果要修改多条相同的文档，则需要设置 multi 参数为 true。例子：`db.col.update({'title':'MongoDB 教程'},{$set:{'title':'MongoDB'}},{multi:true})`
- save() 方法通过传入的文档来替换已有文档。

在创建集合的时候，`db.createCollection(name, options)` options有个可选参数`autoIndexId`,如为 true，自动在 \_id 字段创建索引。默认为 false。(所以mongodb自行创建的集合，默认是没有索引的！)

## 1.4 索引
参考：[https://www.runoob.com/mongodb/mongodb-indexing.html](https://www.runoob.com/mongodb/mongodb-indexing.html)
MongoDB使用 createIndex() 方法来创建索引。
```
db.collection.createIndex(keys, options)
```
语法中 Key 值为你要创建的索引字段，1 为指定按升序创建索引，如果你想按降序来创建索引指定为 -1 即可。

创建联合索引：
```
db.users.ensureIndex({gender:1,user_name:1})
```

### 1.4.1 expireAfterSeconds
`expireAfterSeconds`是options的一个可选参数，类型为integer，用于指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。过期后，MongoDB 独立线程去清除数据。类似于设置定时自动删除任务，可以清除历史记录或日志等前提条件。

有一种用法：由记录中设定日期点清除。即当设置了该字段值后，该条记录将会被过期清理。

例如：
设置 A 记录在 2019 年 1 月 22 日晚上 11 点左右删除，A 记录中需添加 "ClearUpDate": new Date('Jan 22, 2019 23:00:00')，且 Index中expireAfterSeconds 设值为 0。

`db.col.createIndex({"ClearUpDate": 1},{expireAfterSeconds: 0})`

其他注意事项:

- 索引关键字段必须是 Date 类型。
- 非立即执行：扫描 Document 过期数据并删除是独立线程执行，默认 60s 扫描一次，删除也不一定是立即删除成功。
- 单字段索引，混合索引不支持。

### 1.4.2 覆盖索引
因为索引存在于RAM中，从索引中获取数据比通过扫描文档读取数据要快得多。

索引也包含数据，如果要查询的字段都在索引里（或包含），那就达到了索引覆盖的条件，这时直接返回内存中的索引数据即可。

但是，由于我们的索引中不包括 \_id 字段，\_id在查询中会默认返回，所以，需要在查询结果集中排除它。(这是个陷阱)
```
db.users.find({gender:"M"},{user_name:1,_id:0})
```

### 1.4.3 索引数组字段
如果在数组字段上创建了索引，那么会对数组中的每个元素都创建索引。

### 1.4.4 索引子文档字段
也可以在子文档的某个字段上创建索引。

### 1.4.5 索引限制
每个索引占据一定的存储空间，在进行插入，更新和删除操作时也需要对索引进行操作。所以，如果你很少对集合进行读取操作，建议不使用索引。

由于索引是存储在内存(RAM)中,你应该确保该索引的大小不超过内存的限制。

如果索引的大小大于内存的限制，MongoDB会删除一些索引，这将导致性能下降。

**查询限制**
索引不能被以下的查询使用：

- 正则表达式及非操作符，如 $nin, $not, 等。
- 算术运算符，如 $mod, 等。
- $where 子句

所以，检测你的语句是否使用索引是一个好的习惯，可以用explain来查看。

**最大范围**

- 集合中索引不能超过64个
- 索引名的长度不能超过128个字符
- 一个复合索引最多可以有31个字段

### 1.4.6 Best practices
大集合拆分：比如一个用于存储log的collection，log分为有两种“dev”、“debug”，结果大致为{"log":"dev","content":"...."},{"log":"debug","content":"....."}。这两种日志的document个数比较接近，对于查询时，即使给log字段建立索引，这个索引也不是高效的，所以可以考虑将它们分别放在2个Collection中，比如：log_dev和log_debug。

## 1.5 管道
在聚合函数一节中，提到了管道的概念。

MongoDB的聚合管道将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理。管道操作是可以重复的。

表达式：处理输入文档并输出。表达式是无状态的，只能用于计算当前聚合管道的文档，不能处理其它的文档。

这里我们介绍一下聚合框架中常用的几个操作：

- $project：修改输入文档的结构。可以用来重命名、增加或删除域，也可以用于创建计算结果以及嵌套文档。
- $match：用于过滤数据，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit：用来限制MongoDB聚合管道返回的文档数。
- $skip：在聚合管道中跳过指定数量的文档，并返回余下的文档。
- $unwind：将文档中的某一个数组类型字段拆分成多条，每条包含数组中的一个值。
- $group：将集合中的文档分组，可用于统计结果。
- $sort：将输入文档排序后输出。
- $geoNear：输出接近某一地理位置的有序文档。

$match示例：

```
db.articles.aggregate( [
                        { $match : { score : { $gt : 70, $lte : 90 } } },
                        { $group: { _id: null, count: { $sum: 1 } } }
                       ] );
```
$match用于获取分数大于70小于或等于90记录，然后将符合条件的记录送到下一阶段$group管道操作符进行处理。

## 1.6 MongoDB 查询分析
MongoDB 查询分析常用函数有：explain() 和 hint()。

explain 操作提供了查询信息，使用索引及查询统计等。有利于我们对索引的优化。
如果返回结果中indexOnly字段为 true ，表示我们使用了索引。

使用 hint()
虽然MongoDB查询优化器一般工作的很不错，但是也可以使用 hint 来强制 MongoDB 使用一个指定的索引。

这种方法某些情形下会提升性能。 一个有索引的 collection 并且执行一个多字段的查询(一些字段已经索引了)。

如下查询实例指定了使用 gender 和 user_name 索引字段来查询：
```
>db.users.find({gender:"M"},{user_name:1,_id:0}).hint({gender:1,user_name:1})
```
可以使用 explain() 函数来分析以上查询：
```
>db.users.find({gender:"M"},{user_name:1,_id:0}).hint({gender:1,user_name:1}).explain()
```

## 1.7 MongoDB 原子操作
mongodb不支持事务，所以，在你的项目中应用时，要注意这点。无论什么设计，都不要要求mongodb保证数据的完整性。

但是mongodb提供了许多原子操作，比如文档的保存，修改，删除等，都是原子操作。

- 可以使用 db.collection.findAndModify() 方法来判断数据是否已存在并更新数据。
- 原子操作常用命令：$set、$unset等。

## 1.8 全文检索
全文检索对每一个词建立一个索引，指明该词在文章中出现的次数和位置。

比如在文章内容（post_text）上创建全文索引：

创建全文索引：
```
db.posts.ensureIndex({post_text:"text"})
```

使用全文索引:
```
db.posts.find({$text:{$search:"runoob"}})
```

# 2. 部署架构
## 2.1 复制（副本集）
MongoDB复制是将数据同步在多个服务器的过程。

复制提供了数据的冗余备份，并在多个服务器上存储数据副本，提高了数据的可用性， 并可以保证数据的安全性。

复制还允许您从硬件故障和服务中断中恢复数据。

mongodb的架构方式之一 ，通常是三个对等的节点构成一个“复制集”集群，有“primary”和secondary等多中角色（稍后详细介绍），其中primary负责读写请求，secondary可以负责读请求，这有配置决定，其中secondary紧跟primary并应用write操作；如果primay失效，则集群进行“多数派”选举，选举出新的primary，即failover机制，即HA架构。复制集解决了单点故障问题，也是mongodb垂直扩展的最小部署单位，当然sharding cluster中每个shard节点也可以使用Replica set提高数据可用性。

### 2.1.1 MongoDB复制原理
mongodb的复制至少需要两个节点。其中一个是主节点，负责处理客户端请求，其余的都是从节点，负责复制主节点上的数据。

mongodb各个节点常见的搭配方式为：一主一从、一主多从。

主节点记录在其上的所有操作oplog，从节点定期轮询主节点获取这些操作，然后对自己的数据副本执行这些操作，从而保证从节点的数据与主节点一致。

MongoDB的副本集与我们常见的主从有所不同，主从在主机宕机后所有服务将停止，而副本集在主机宕机后，副本会接管主节点成为主节点，不会出现宕机的情况。

## 2.2 分片（sharing）
在Mongodb里面存在另一种集群，就是分片技术,可以满足MongoDB数据量大量增长的需求。数据水平扩展的手段之一。

replica set这种架构的缺点就是“集群数据容量”受限于单个节点的磁盘大小，如果数据量不断增加，对它进行扩容将时非常苦难的事情，所以我们需要采用Sharding模式来解决这个问题。

当MongoDB存储海量的数据时，一台机器可能不足以存储数据，也可能不足以提供可接受的读写吞吐量。这时，我们就可以通过在多台机器上分割数据，使得数据库系统能存储和处理更多的数据。

下图展示了在MongoDB中使用分片集群结构分布：
![pic](https://www.runoob.com/wp-content/uploads/2013/12/sharding.png)

上图中主要有如下所述三个主要组件：

- Shard:用于存储实际的数据块，实际生产环境中一个shard server角色可由几台机器组成一个replica set承担，防止主机单点故障

- Config Server:mongod实例，存储了整个 ClusterMetadata，其中包括 chunk信息。

- Query Routers:前端路由，客户端由此接入，且让整个集群看上去像单一数据库，前端应用可以透明使用。


# 3. MongoDB GridFS
GridFS 用于存储和恢复那些超过16M（BSON文件限制）的文件(如：图片、音频、视频等)。

GridFS 也是文件存储的一种方式，但是它是存储在MonoDB的集合中。

GridFS 可以更好的存储大于16M的文件。

GridFS 会将大文件对象分割成多个小的chunk(文件片段),一般为256k/个,每个chunk将作为MongoDB的一个文档(document)被存储在chunks集合中。

GridFS 用两个集合来存储一个文件：fs.files与fs.chunks。
每个文件的实际内容被存在chunks(二进制数据)中,和文件有关的meta数据(filename,content_type,还有用户自定义的属性)将会被存在files集合中。