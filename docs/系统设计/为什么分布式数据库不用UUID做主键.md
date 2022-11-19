# 为什么分布式数据库不用UUID做主键

1. UUID生成速率低下

Java的UUID依赖于SecureRandom.nextBytes方法，而SecureRandom又依赖于操作系统提供的随机数源，在Linux系统下，它的默认依赖是/dev/random，而这个源是阻塞的。最可怕的是，这个nextBytes方法还是一个synchronized方法，也就是说，如果多线程调用UUID，生成速率不升反降。

测试结果：在一台64线程的服务器上，调用UUID.randomUUID方法，生成一千万个uuid平均耗时在130s，tps不到8w

 

2. UUID主键在innodb中会引发性能问题

a. innodb中的主键索引也是聚集索引，如果插入的数据是顺序的，那么b+树的叶子基本都是满的，缓存也可以很好的发挥作用。如果插入的数据是完全无序的，那么叶子节点会频繁分裂，缓存也基本无效了。这会减少tps

b. uuid占用的空间较大

 

3. UUID完全没有意义，如果有一个主键是全局自增的，那么数据排列顺序就是数据的插入顺序

 

解决方案：

1. 分布式全局序列生成（使用zk的DistributedAtomicLong，一次自增一个步长，用户用完了步长内的序列，再找zk要）

2. Twitter的snowflake算法

当然自增序列也不是完美的，因为在极大并发的情况下，按自增主键插入会发生争用，主键的上界会出现热点。但总的来说，还是可以接受的