# innodb事务的底层实现技术

## mysql的一条数据是如何最后写入的

1. 开始执行事务,获取相关锁

2. 记录undolog (保证事务原子性,先写undolog回滚日志,逻辑日志,记录在ibdata1文件, 理解为修改数据之前备份在别的地方去, 也用于实现MVCC)
    
    **MVCC(多版本控制 Multi version Currency Control)**
    
    一般情况下，事务性储存引擎不是只使用表锁，行加锁的处理数据，而是结合了MVCC机制，以处理更多的并发问题。
    
3. 修改数据,在内存中修改 (未提交,属于脏事务)

4. 记录redolog (保证事务持久性,INNODB记录redo log 重做日志,redolog为物理日志,对应文件为ib_logfileN,redolog是顺序循环写,相对高效。在事务进行中持续写)

5. 事务提交。提交完成并不代表数据不会丢失,这与上面配置的参数有关。同时,写binlog,为了保证master和slave的数据一致性和崩溃恢复,就必须保证binlog和INNODB redolog的一致性（binlog在server层,而redo log 在存储引擎层）,MySQL引入二阶段提交,传说中的2PC。

```
以上,不管是redolog,undolog,binlog,都是先写日志缓冲,再写磁盘,对应参数innodb_flush_log_at_trx_commit ,sync_binlog。
```

6. 数据落盘,与参数配置有关,配置的严格 双1模式下e步骤就可落盘,一般高性能模式是会延迟落盘,不会立即落盘。
![](/uploads/upload_a6f18295195b10f5122be5b0e99bc1d2.png)


## 事务基本特性 ACID

原子性（Atomicity）：指的是一个事务中的操作要么全部成功，要么全部失败。

一致性（Consistency）：指的是数据库总是从一个一致性的状态转换到另外一个一致性的状态。

隔离性（Isolation）：指的是一个事务的修改在最终提交前，对其他事务是不可见的。

持久性（Durability）：指的是一旦事务提交，所做的修改就会永久保存到数据库中。

## 事务隔离级别：

- read uncommit 读未提交，可能会读到其他事务未提交的数据，也叫做脏读。

    用户本来应该读取到id=1的用户age应该是10，结果读取到了其他事务还没有提交的事务，结果读取结果age=20，这就是脏读。
    
![](/uploads/upload_9dd0247b33acc33b8951ea29b9c48c71.png)


- read commit 读已提交，两次读取结果不一致，叫做不可重复读。

    不可重复读解决了脏读的问题，他只会读取已经提交的事务。

    用户开启事务读取id=1用户，查询到age=10，再次读取发现结果=20，在同一个事务里同一个查询读取到不同的结果叫做不可重复读。

![](/uploads/upload_bc4f97796f7a1d2f540855ff97da6e87.png)

- repeatable read 可重复复读，这是mysql的默认级别，就是每次读取结果都一样，但是有可能产生**幻读**。

- serializable 串行，一般是不会使用的，他会给每一行读取的数据加锁，会导致大量超时和锁竞争的问题。

## ACID怎么保证的

- A原子性： 由undo log日志保证，它记录了需要回滚的日志信息，事务回滚时撤销已经执行成功的sql

- C一致性： 原子性、隔离性、持久性折腾半天就是为了保证数据的一致性。

- I隔离性： 由MVCC来保证

- D持久性： 由内存+redo log来保证，mysql修改数据同时在内存和redo log记录这次操作，事务提交的时候通过redo log刷盘，宕机的时候可以从redo log恢复。