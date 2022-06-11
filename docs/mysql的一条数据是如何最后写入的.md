# mysql的一条数据是如何最后写入的

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
