# 2pc两阶段提交

**保证binlog和redo log的一致性。**

因为binlog是Master-Slave的桥梁，如果顺序不一致，意味着Master-Slave可能不一致。MYSQL通过两阶段提交很好地解决了这一问题。Prepare阶段，innodb刷redo log，并将回滚段设置为Prepared状态，binlog不作任何操作；commit阶段，innodb释放锁，释放回滚段，设置提交状态，binlog刷binlog日志。出现异常，需要故障恢复时，若发现事务处于Prepare阶段，并且binlog存在则提交，否则回滚。通过两阶段提交，保证了redo log和binlog在任何情况下的一致性