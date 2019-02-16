 ## InnoDB-一致性非锁定读（快照读）
     mysql官网文档：If the transaction isolation level is REPEATABLE READ (the default level), all consistent reads within the same transaction read 
     the snapshot established by the first such read in that transaction. You can get a fresher snapshot for your queries by committing the current 
     transaction and after that issuing new queries.
     翻译：如果事务隔离级别是 REPEATABLE READ (默认级别)，同一个事务中的所有一致性读都会读取此事务中第一次一致性读所生成的快照。你可以通过提交当前事务然后进行一个新的查询来得到一个最新的查询快照。
     
 前面有说到next-key lock 和快照读，当时是不确定是next-key lock是保证不出现幻读的关键还是快照读是关键，他们的关系究竟是怎么样的，看完InnoDB的一致性读之后，就能揭晓答案了。
 
 一致性读取是指：InnoDB使用多版本控制让查询到的数据都是数据库在某一个时间点的快照。查询到的结果是在该时间点之前所提交的事务做出的更改，而不会对以后或未提交的事务所做的更改进行更新。
 为什么叫未锁定读？
 
 因为在读取数据的时候，不需要等待访问的行上X锁的释放。而是直接读取快照。如图。
 那么，快照数据又是什么，存在哪的，怎么存的？
 
 ![](./img/transaction/一致性读.png)
 
    
参考：<https://www.cnblogs.com/zhiqian-ali/p/5668199.html> 《MySQL技术内幕》
      <https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html> 《Mysql官网文档》

       