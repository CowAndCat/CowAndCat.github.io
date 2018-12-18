---
layout: post
title: 分布式事务之TCC事务
category: arch
comments: false
---
## 一、TCC 事务介绍
目前很多分布式事务开源项目也都是基于TCC的思想实现。

TCC型事务（Try/Confirm/Cancel）可以归为补偿型。补偿型的例子，在一个长事务（ long-running ）中 ，一个由两台服务器一起参与的事务，服务器A发起事务，服务器B参与事务，B的事务需要人工参与，所以处理时间可能很长。如果按照ACID的原则，要保持事务的隔离性、一致性，服务器A中发起的事务中使用到的事务资源将会被锁定，不允许其他应用访问到事务过程中的中间结果，直到整个事务被提交或者回滚。这就造成事务A中的资源被长时间锁定，系统的可用性将不可接受。

TCC 将事务提交分为 Try - Confirm - Cancel 3个操作。

- Try：预留业务资源/数据效验
- Confirm：确认执行业务操作
- Cancel：取消执行业务操作

TCC事务处理流程和 2PC 二阶段提交类似，不过 2PC通常都是在跨库的DB层面，而TCC本质就是一个应用层面的2PC。

![tcc](/images/201812/tcc-theory.jpg)

TCC原理图，图片来自阿里公众号文章

## 二、TCC 优缺点
TCC优点：让应用自己定义数据库操作的粒度，使得降低锁冲突、提高吞吐量成为可能。

TCC不足之处：

- 对应用的侵入性强。业务逻辑的每个分支都需要实现try、confirm、cancel三个操作，应用侵入性较强，改造成本高。
- 实现难度较大。需要按照网络状态、系统故障等不同的失败原因实现不同的回滚策略。为了满足一致性的要求，confirm和cancel接口必须实现幂等。

## 三、TCC 事务应用场景
我们通过用户下单使用余额+红包支付来看一下TCC事务的具体应用。

假设用户下单操作来自3个系统下单系统、资金账户系统、红包账户系统，下单成功需要同时调用资金账户服务和红包服务完成支付

假设购买商品1000元，使用账户红包200元，余额800元，确认支付。

- Try操作

        tryX 下单系统创建待支付订单
        tryY 冻结账户红包200元
        tryZ 冻结资金账户800元

- Confirm操作

        confirmX 订单更新为支付成功
        confirmY 扣减账户红包200元
        confirmZ 扣减资金账户800元

- Cancel操作

    cancelX 订单处理异常，资金红包退回，订单支付失败
    cancelY 冻结红包失败，账户余额退回，订单支付失败
    cancelZ 冻结余额失败，账户红包退回，订单支付失败

## 四、TCC 开源项目实战 tcc-transaction

（实战项目请看原文：https://www.liangzl.com/get-article-detail-525.html）

> tcc-transaction 是TCC型事务Java实现，tcc-transaction不和底层使用的rpc框架耦合，也就是使用doubbo,thrift,web service,http等都可以.

在 tcc-transaction 事例中有dubbo、http实现，在这里我们使用dubbo事例。

事例的具体功能，其实就是上面介绍的 下单使用余额+红包支付。

tcc-transaction 事务管理器日志持久化支持多种方式

> 引用官方1.2指南说明：tcc-transaction框架使用transactionRepository持久化事务日志。可以选择FileSystemTransactionRepository、SpringJdbcTransactionRepository、RedisTransactionRepository或ZooKeeperTransactionRepository。

我们这里演示使用SpringJdbcTransactionRepository，存储到MySQL

### 4.1 项目准备
假设本地环境MySQL、Zookeeper都已经准备完毕

使用IDEA，将项目克隆到本地

我们使用的实例为tcc-transaction-dubbo-sample

- tcc-transaction-dubbo-capital 资金账户dubbo服务
- tcc-transaction-dubbo-capital-api 资金账户dubbo接口定义
- tcc-transaction-dubbo-order 下单系统
- tcc-transaction-dubbo-redpacket 红包账户dubbo服务
- tcc-transaction-dubbo-redpacket-api 红包账户dubbo接口定义

执行数据库脚本： 

- tcc-transaction\tcc-transaction-tutorial-sample\src\dbscripts\create_db_cap.sql (资金账户)
- tcc-transaction\tcc-transaction-tutorial-sample\src\dbscripts\create_db_ord.sql (订单库)
- tcc-transaction\tcc-transaction-tutorial-sample\src\dbscripts\create_db_red.sql (红包账户)
- tcc-transaction\tcc-transaction-tutorial-sample\src\dbscripts\create_db_tcc.sql (事务日志持久化)

修改项目配置： Zookeeper 配置

    zookeeper.address=127.0.0.1:2181

如果非默认配置，需要将tcc-transaction-dubbo-sample 下的模块全部修改

JDBC连接配置：

    jdbc.driverClassName=com.mysql.jdbc.Driver
    tcc.jdbc.url=jdbc:mysql://127.0.0.1:3306/TCC?useUnicode=true&characterEncoding=UTF-8&allowMultiQueries=true
    jdbc.username=root
    jdbc.password=

如果非默认配置，需要将tcc-transaction-dubbo-sample，tcc-transaction-sample-domain 下项目模块修改

### 4.2 项目发布
实例目前都是使用的Tomcat发布，我们先部署Tomcat：

- tcc-transaction-dubbo-capital 服务 http://192.168.10.199:8881/
- tcc-transaction-dubbo-order 服务 http://192.168.10.199:8882/
- tcc-transaction-dubbo-capital 服务 http://192.168.10.199:8883/

### 4.3 多种事务场景测试
第一种情况：红包账户冻结成功(try)、资金账户冻结成功(try)，订单操作异常(try)

此时订单支付失败，红包账户冻结金额、资金账户冻结金额全部退回。
通过多种类似方式测试，例如：红包账户冻结异常、资金账户冻结异常，都会调cancel，退回冻结资源。所以在try阶段的任意一方异常，都会执行全局回滚。

第二种情况：订单处理成功(confirm)，资金账户扣减成功(confirm)，但红包账户扣减失败(confirm)

此时订单为支付成功，但实际红包金额还处在冻结状态，事务管理器记录订单confirm操作未执行成功，系统会不断重试调用订单的confirm操作，直到红包扣减成功。

手动将红包服务异常代码去掉，重启服务，等到下一次重试，红包冻结金额被扣除成功。

第三种情况: 资金账户冻结成功(try)，红包账户冻结成功(try)，订单处理失败(confirm)

此时订单支付成功，但实际资金账户冻结金额、红包冻结金额都还没有扣除成功，事务管理器记录订单confirm操作未执行成功，系统会通过不断重试订单的confirm操作，直到资金账户和红包账户扣减成功。

手动将异常代码去掉，重启服务，等到下一次重试，红包和资金账户冻结金额被扣除。

第四种情况：在第一种情况下，订单cancel操作处理失败(cancel)

此时订单支付失败，资金账户、红包账户冻结成功，事务管理器记录订单cancel操作失败，系统会不断重试订单的cancel操作，直到资金账户和红包账户冻结金额退回账户。

手动将异常代码去掉，重启服务，下一次重试cancel操作，资金账户和红包账户冻结金额退回。

### 4.4 总结
try 操作成功，进入 confirm 操作，只要 confirm 处理失败（不管是协调者挂了，还是参与者处理失败或超时），系统通过不断重试直到处理成功。 进入 cancel 操作也是一样，只要 cancel 处理失败，系统通过不断重试直到处理成功。

引用一段对话，具体说明TCC使用场景，以及和2PC对比：

问题

- 1.对于转账的cancel事务，相对简单，只要把账务冲负即可。可一般的业务逻辑会涉及很多流程、单证等操作，尤其是历史系统，应该很难改造成tcc结构的吧，不知道你们用tcc用在什么场景下？
- 2.我第一次是在支付宝架构师程立PPT中听说tcc结构的，不知道你的实现跟阿里系的实现是什么关系，目前应用的规模大吗？
- 3.我理解try程序完成后，立即提交try事务，不会有锁事务竞争。可这个时候账户余额的状态也被设置成了类似不可读的状态吧，依然不可以在其他业务中查询账户余额，那么这种方案比2PC优势具体在什么地方？

回答

- 1.我的感觉TCC是比较适合具有较强一致性要求的场景，账务系统就是这样场景；应用TCC也有一定的开发成本，如果没有强一致性要求，可以考虑其他补偿型方案；
- 2.目前已应用在线上环境多个应用，主要就是应用账务系统里使用；因为不是开源的，我没有看过阿里的实现，部分借鉴了他们的思路；希望有机会能看看；
- 3.在Try结束后，账户余额会有部分资金冻结，其他业务不可以使用冻结资金；和2PC比较的话，理解下来有两点：
    - 2PC是基于资源层（如数据库），TCC是基于SOA服务
    - 2PC是是用全局事务，数据被lock的时间跨整个事务，直到全局事务结束；而TCC里每个对资源操作的是本地事务，数据被lock的时间短，可扩展性好.


# REF
> [分布式事务之TCC事务](https://www.liangzl.com/get-article-detail-525.html)