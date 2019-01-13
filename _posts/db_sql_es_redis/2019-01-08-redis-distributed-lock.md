---
layout: post
title: Redis实现分布式锁
category: redis
comments: false
---

## 一、Redis分布式锁
分布式锁一般有三种实现方式：1. 数据库乐观锁；2. 基于Redis的分布式锁；3. 基于ZooKeeper的分布式锁。本篇博客将介绍第二种方式，基于Redis实现分布式锁。

### 1.1 实现
加锁方法：`SET key random_value EX 10086 NX`  
如果不存在就设置value并加上过期时间，否则返回false。

value 是拥有者标识，只有 key-value 匹配才可以解锁;value的值必须是随机数主要是为了更安全的释放锁，释放锁的时候使用脚本告诉Redis:只有key存在并且存储的值和我指定的值一样才能告诉我删除成功。

举个例子：客户端A取得资源锁，但是紧接着被一个其他操作阻塞了，当客户端A运行完毕其他操作后要释放锁时，原来的锁早已超时并且被Redis自动释放，并且在这期间资源锁又被客户端B再次获取到。如果仅使用DEL命令将key删除，那么这种情况就会把客户端B的锁给删除掉。如果value采用random，脚本仅会删除value等于客户端A的value的key（value相当于客户端的一个签名）。

设置过期时间是为了防止锁的持有者后续发生崩溃而没有解锁从而发生死锁的情况；
set 和设置时间的操作必须保证原子性（不能将setNX和setExpire分开），否则 set 后程序突然崩溃就无法设置过期时间，将发生死锁。

解锁方法：用 Lua 脚本执行 
`if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end`

即判断锁拥有者和解锁操作同样要保持原子性，否则可能会解除其他人的锁。
eval 命令执行 Lua 代码的时候，Lua 代码将被当成一个命令去执行，并且直到 eval 命令执行完成，Redis 才会执行其他命令。


### 1.2 Redisson代码实现

    String key = "key";
    RLock lock = redissonClient.getLock(key);
    lock.lock(60, TimeUnit.SECONDS); //设置60秒自动释放锁  （默认是30秒自动过期）
    try {
        // do something.
    } catch (Exception e) {
        lock.unlock();
    }

## 二、不足
为什么不建议使用 redis 分布锁？

## 2.1 主从切换可能丢失锁信息

考虑一下这样的场景：在分布式环境中，很多并发需要锁来同步，当使用 redis 分布式锁，通用的做法是使用 redis 的 SET key value EX 10086 NX 这样的命令，设置一个字段，当设置成功说明获取锁，设置不成功说明锁被占用，当获取所之后需要删除锁，也就是删除设置的锁字段，这是锁可以被其他占用。

这里在主从切换回出现问题，当第一个线程在主服务器上设置了锁，但是这时候从服务器并没有及时同步主服务器的状态，也就是没有同步主服务器中的锁字段，而此时，主服务器挂了，redis 的哨兵模式升级从服务器为主服务器，如果在并发量大的情况下，虽然第一个线程获取了锁，其他线程会在当前的主服务器（之前的从服务器，但是并没有同步已经设置的锁字段）上设置锁字段，这样并不能保证锁的互斥性。

这个缺陷可被 zookeeper 的单调一致性弥补。或是基于RedLock来实现。

## 2.2 缓存易失性
假如第一个线程设置了锁，但是之后触发内存淘汰机制很不幸淘汰了设置的锁字段，接下来的线程在第一个线程没有释放锁的情况下，也是重新设置锁字段的，这样并不能保证锁的安全性。

## REF
> [Redis分布式锁的正确实现方式](https://www.cnblogs.com/linjiqin/p/8003838.html)  
> [Redis应用五：Redisson](https://www.jianshu.com/p/6f7d6a1c3bc2)  
> [jedisLock—redis分布式锁实现](http://www.cnblogs.com/0201zcr/p/5942748.html)