---
layout: post
title: MongoDB存储特性与内部原理
category: mongodb
comments: false
---
# 一、存储引擎（Storage）

mongodb 3.0默认存储引擎为MMAPV1，还有一个新引擎wiredTiger可选(官方宣称在read、insert和复杂的update下具有更高的性能)，或许可以提高一定的性能。

备注：mongodb 3.2+之后，默认的存储引擎为“wiredTiger”，大量优化了存储性能，建议升级到3.2+版本。

mongodb中有多个databases，每个database可以创建多个collections，collection是底层数据分区（partition）的单位，每个collection都有多个底层的数据文件组成。（参见下文data files存储原理）

## 1.1 MMAPV1引擎
### 1.1.1 Data Files

mongodb的数据将会保存在底层文件系统中，比如我们dbpath设定为“/data/db”目录，我们创建一个database为“test”，collection为“sample”，然后在此collection中插入数条documents。我们查看dbpath下生成的文件列表：(test.0-test.6，外加一个test.ns )

test.0-test.6 为6个数据文件。每个文件以“database”的名字 + 序列数字组成，序列号从0开始，逐个递增，数据文件从16M开始，每次扩张一倍（16M、32M、64M、128M...），在默认情况下单个data file的最大尺寸为2G，如果设置了smallFiles属性（配置文件中）则最大限定为512M；mongodb中每个database最多支持16000个数据文件，即约32T，如果设置了smallFiles则单个database的最大数据量为8T。

一个database中所有的collections以及索引信息会分散存储在多个数据文件中，即mongodb并没有像SQL数据库那样，每个表的数据、索引分别存储；数据分块的单位为extent（范围，区域），即一个data file中有多个extents组成，extent中可以保存collection数据或者indexes数据，一个extent只能保存同一个collection数据，不同的collections数据分布在不同的extents中，indexes数据也保存在各自的extents中；最终，一个collection有一个或者多个extents构成，最小size为8K，最大可以为2G，依次增大；它们分散在多个data files中。对于一个data file而言，可能包含多个collection的数据，即有多个不同collections的extents、index extents混合构成。每个extent包含多条documents（或者index entries），每个extent的大小可能不相等，但一个extent不会跨越2个data files。

有人肯定疑问：一个collection中有哪些extents，这种信息mongodb存在哪里？在每个database的namespace文件中，比如test.ns文件中，每个collection只保存了第一个extent的位置信息，并不保存所有的extents列表，但每个extent都维护者一个链表关系，即每个extent都在其header信息中记录了此extent的上一个、下一个extent的位置信息，这样当对此collection进行scan操作时（比如全表扫描），可以提供很大的便利性。

可以通过db.stats()指令查看当前database中extents的信息。

删除document会导致磁盘碎片，有些update也会导致磁盘碎片，比如update导致文档尺寸变大，进而超过原来分配的空间；当有新的insert操作时，mongodb会检测现有的extents中是否合适的碎片空间可以被重用，如果有，则重用这些fragment，否则分配新的存储空间。磁盘碎片，对write操作有一定的性能影响，而且会导致磁盘空间浪费；如果你需要删除某个collection中大部分数据，则可以考虑将有效数据先转存到新的collection，然后直接drop()原有的collection。或者使用`db.runCommand({compact: '<collection>'})`。

如果你的database已经运行一段时间，数据已经有很大的磁盘碎片（storageSize与dataSize比较），可以通过mongodump将指定database的所有数据导出，然后将原有的db删除，再通过mongorestore指令将数据重新导入。（同compact，这种操作需要停机维护）

### 1.1.2 Namespace文件
 对于namespace文件，比如“test.ns”文件，默认大小为16M，此文件中主要用于保存“collection”、index的命名信息，比如collection的“属性”信息、每个索引的属性类型等，如果你的database中需要存储大量的collection（比如每一小时生成一个collection，在数据分析应用中），那么我们可以通过配置文件“nsSize”选项来指定。

### 1.1.3 journal文件

journal日志为mongodb提供了数据保障能力，它本质上与mysql binlog没有太大区别，用于当mongodb异常crash后，重启时进行数据恢复；这归结于mongodb的数据持久写入磁盘是滞后的。默认情况下，“journal”特性是开启的，特别在production环境中，我们没有理由来关闭它。（除非，数据丢失对应用而言，是无关紧要的）

一个mongodb实例中所有的databases共享journal文件。

对于write操作而言，首先写入journal日志，然后将数据在内存中修改（mmap），此后后台线程间歇性的将内存中变更的数据flush到底层的data files中，时间间隔为60秒（参见配置项“syncPeriodSecs”）；write操作在journal文件中是有序的，为了提升性能，write将会首先写入journal日志的内存buffer中，当buffer数据达到100M或者每隔100毫秒，buffer中的数据将会flush到磁盘中的journal文件中；如果mongodb异常退出，将可能导致最多100M数据或者最近100ms内的数据丢失，flush磁盘的时间间隔有配置项“commitIntervalMs”决定，默认为100毫秒。mongodb之所以不能对每个write都将journal同步磁盘，这也是对性能的考虑，mysql的binlog也采用了类似的权衡方式。

开启journal日志功能，将会导致write性能有所降低，可能降低5~30%，因为它直接加剧了磁盘的写入负载，我们可以将journal日志单独放置在其他磁盘驱动器中来提高写入并发能力（与data files分别使用不同的磁盘驱动器）。

如果你希望数据尽可能的不丢失，可以考虑：

1）减小commitIntervalMs的值  
2）每个write指定“write concern”中指定“j”参数为true   
3）最佳手段就是采用“replica set”架构模式，通过数据备份方式解决，同时还需要在“write concern”中指定“w”选项，且保障级别不低于“majority”。

根据write并发量，journal日志文件为1G，如果指定了smallFiles配置项，则最大为128M，和data files一样journal文件也采用了“preallocated”方式，journal日志保存在dbpath下“journal”子目录中，一般会有三个journal文件，每个journal文件格式类似于“j.\_<序列数字>”。并不是每次buffer flush都生成一个新的journal日志，而是当前journal文件即将满时会预创建一个新的文件，journal文件中保存了write操作的记录，每条记录中包含write操作内容之外，还包含一个“lsn”（last sequence number），表示此记录的ID；此外我们会发现在journal目录下，还有一个“lsn”文件，这个文件非常小，只保存了一个数字，当write变更的数据被flush到磁盘中的data files后，也意味着这些数据已经持久化了，那么它们在“异常恢复”时也不需要了，那么其对应的journal日志将可以删除，“lsn”文件中记录的就是write持久化的最后一个journal记录的ID，此ID之前的write操作已经被持久写入data files，此ID之前的journal在“异常恢复”时则不需要关注；如果某个journal文件中最大 ID小于“lsn”，则此journal可以被删除或者重用。

### 1.1.4 总结MMAPv1引擎

mongodb原生的存储引擎，比较简单，直接使用系统级的内存映射文件机制（memory mapped files），一直是mongodb的默认存储引擎，对于insert、read和in-place update（update不导致文档的size变大）性能较高；不过MMAPV1在lock的并发级别上，支持到collection级别，所以对于同一个collection同时只能有一个write操作执行，这一点相对于wiredTiger而言，在write并发性上就稍弱一些。对于production环境而言，较大的内存可以使此引擎更加高效，有效减少“page fault”频率，但是因为其并发级别的限制，多核CPU并不能使其受益。此引擎将不会使用到swap空间，但是对于wiredTiger而言需要一定的swap空间。（核心：对于大文件MAP操作，比较忌讳的就是在文件的中间修改数据，而且导致文件长度增长，这会涉及到索引引用的大面积调整）

为了确保数据的安全性，mongodb将所有的变更操作写入journal并间歇性的持久到磁盘上，对于实际数据文件将延迟写入，和wiredTiger一样journal也是用于数据恢复。所有的记录在磁盘上连续存储，当一个document尺寸变大时，mongodb需要重新分配一个新的记录（旧的record标记删除，新的记record在文件尾部重新分配空间），这意味着mongodb同时还需要更新此文档的索引（指向新的record的offset），与in-place update相比，将消耗更多的时间和存储开支。由此可见，如果你的mongodb的使用场景中有大量的这种update，那么或许MMAPv1引擎并不太适合，同时也反映出如果document没有索引，是无法保证document在read中的顺序（即自然顺序）。3.0之后，mongodb默认采用“Power of 2 Sized Allocations”，所以每个document对应的record将有实际数据和一些padding组成，这padding可以允许document的尺寸在update时适度的增长，以最小化重新分配record的可能性。此外重新分配空间，也会导致磁盘碎片（旧的record空间）。

## 1.2 wiredTiger引擎
所有的write请求都基于“文档级别”的lock，因此多个客户端可以同时更新一个colleciton中的不同文档，这种更细颗粒度的lock，可以支撑更高的读写负载和并发量。因为对于production环境，更多的CPU可以有效提升wireTiger的性能，因为它是的IO是多线程的。wiredTiger不像MMAPV1引擎那样尽可能的耗尽内存，它可以通过在配置文件中指定“cacheSizeGB”参数设定引擎使用的内存量，此内存用于缓存工作集数据（索引、namespace，未提交的write，query缓冲等）。

journal就是一个预写事务日志，来确保数据的持久性，wiredTiger每隔60秒（默认）或者待写入的数据达到2G时，mongodb将对journal文件提交一个checkpoint（检测点，将内存中的数据变更flush到磁盘中的数据文件中，并做一个标记点，表示此前的数据表示已经持久存储在了数据文件中，此后的数据变更存在于内存和journal日志）。

对于write操作，首先被持久写入journal，然后在内存中保存变更数据，条件满足后提交一个新的检测点，即检测点之前的数据只是在journal中持久存储，但并没有在mongodb的数据文件中持久化，延迟持久化可以提升磁盘效率，如果在提交checkpoint之前，mongodb异常退出，此后再次启动可以根据journal日志恢复数据。

journal日志默认每个100毫秒同步磁盘一次，每100M数据生成一个新的journal文件，journal默认使用了snappy压缩，检测点创建后，此前的journal日志即可清除。mongod可以禁用journal，这在一定程度上可以降低它带来的开支；对于单点mongod，关闭journal可能会在异常关闭时丢失checkpoint之间的数据（那些尚未提交到磁盘数据文件的数据）；对于replica set架构，持久性的保证稍高，但仍然不能保证绝对的安全（比如replica set中所有节点几乎同时退出时）。


# 二、Capped Collections
一种特殊的collection，其尺寸大小是固定值，类似于一个可循环使用的buffer，如果空间被填满之后，新的插入将会覆盖最旧的文档，我们通常不会对Capped进行删除或者update操作，所以这种类型的collection能够支撑较高的write和read，通常情况下我们不需要对这种collection构建索引，因为insert是append（insert的数据保存是严格有序的）、read是iterator方式，几乎没有随机读；在replica set模式下，其oplog就是使用这种colleciton实现的。 

Capped Collection的设计目的就是用来保存“最近的”一定尺寸的document。 

Capped Collection在语义上，类似于“FIFO”队列，而且是有界队列。适用于数据缓存，消息类型的存储。Capped支持update，但是我们通常不建议，如果更新导致document的尺寸变大，操作将会失败，只能使用in-place update，而且还需要建立合适的索引。在capped中使用remove操作是允许的。autoIndex属性表示默认对_id字段建立索引，我们推荐这么做。在上文中我们提到了Tailable Cursor，就是为Capped而设计的，效果类似于“tail -f ”。

# REF
>[Mongodb存储特性与内部原理](https://www.iteye.com/blog/shift-alt-ctrl-2255580)  
>[Mongodb中Index特性](https://www.iteye.com/blog/shift-alt-ctrl-2250101)