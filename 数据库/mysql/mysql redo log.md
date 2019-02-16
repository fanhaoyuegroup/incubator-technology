
# redo log 文件

## 什么是redo log
> * 记录了对MYSQL数据库innodb存储引擎,记录了innodb存储引擎的事务日志,用于恢复数据。每个innodb存储引擎至少有一个redo log文件组,每个文件组中至少有两个redo log文件,每个文件大小一致,并以循环写入的方式运行.并且redo log不是直接写到磁盘,是先写入redo log缓存,在刷盘到磁盘,当两个文件都写满的时候,会进行checkpoint,将文件1的内容抹除,如图
![](/img/redolog.jpg)

## redo log 和 binlog的区别
### 这里可能大家会有疑问,binlog可以恢复,那么要redo log有什么意义呢

> * redo log 是innodb引擎特有的,binlog是mysql的server层实现的,所有引擎都可以使用
> * redo log 是物理日志,记录了在某个数据页的修改,binlog是逻辑日志,记录的是这条记录的修改逻辑
> * redo log 是循环写的,空间固定会用完,binlog是可以追加写入的,"追加写"是指binlog文件写到一定大小后会切换到下一个,并不会覆盖以前的日志
> * redo log 的大小不易设置的过大或者过小,过小的话会导致频繁的checkpoint导致性能抖动,太大则恢复会需要更长的时间

## 两阶段提交
> * 为了保证恢复数据时的状态准确性,维护逻辑一致性,采用两阶段提交,例如一次更新数据,如图
![](/img/两阶段提交.jpg)
> * 根据binlog的特性,我们可以知道binlog是在事务提交前写入,如果事务还未提交的时候,这个时候binlog已经写入了,但是数据库发生了异常宕机,那么根据binlog来恢复,就会把未提交的事务给恢复到库中,造成脏数据,而采用两阶段提交,这个时候我们看到redo log处于prepare阶段,binlog已经写了日志
则可以对未提交事务的脏数据做处理,防止恢复的时候出现脏数据

## redo log 的写入
> * 通过设置参数innodb_flush_log_at_trx_commit的值来控制将缓存中的redo log写入磁盘,0代表提交事务时,不将redo log写入磁盘,而是等每次主线程刷新,1和2的区别在于,1表示在commit的时候将redo log从缓存中同步到磁盘,即伴有fsync的调用,2表示将redo log异步写到磁盘,即写到文件系统的缓存中,因此不能完全保证执行commit的时候会一定写入磁盘,为了保证事务的ACID中的持久性,必须将此参数设置为1,也就是每当有事务提交时,必须保证日志写入磁盘