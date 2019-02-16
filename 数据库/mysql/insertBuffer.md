#Insert Buffer 

**insert buffer 的架构**
>![](图片.png)
**什么是insert Buffer**
>* 用来提升插入性能(先插入缓存,最后合并,避免非顺序插入造成磁盘离散查询,性能很差)
>* 表的主键是id，id列式自增长的，即当执行插入操作时，id列会自动增长，页中行记录按id顺序存放，不需要随机读取其它页的数据。因此，在这样的情况下（即聚集索引），插入操作效率很高。(不过对于UUID这类型的主键插入也可能是离散的,所以不推荐使用UUID做主键)
>* 除了主键聚合索引外，还产生了一个name列的辅助索引，对于该非聚集索引来说，叶子节点的插入不再有序，这时就需要离散访问非聚集索引页，插入性能变低。


**什么情况下使用insert Buffer**
>
>* 索引是辅助索引
>* 索引是唯一的(如果索引是唯一的,那么插入时需要检验索引的唯一性,还是会直接进行离散IO查询.导致insert Buffer失效)

**什么是change Buffer**
>
>* 类似insert buffer,针对于更新和删除操作
>
>> * 1.缓存中标记删除
>> * 2.文件中真正删除

**insert Buffer内部实现**
>insert buffer的数据结构是一棵B+树。在MySQL4.1之前的版本中每张表都有一棵insert buffer B+树。而在现在的版本中，全局只有一棵insert buffer B+树，负责对所有的表的辅助索引进行 insert buffer。这棵B+树存放在共享表空间中，默认也就是ibdata1中。因此，试图通过独立表空间ibd文件恢复表中数据时，往往会导致check table 失败。这是因为表的辅助索引中的数据可能还在insert buffer中，也就是共享表空间中。所以通过idb文件进行恢复后，还需要进行repair table 操作来重建表上所有的辅助索引。

>insert buffer是一棵B+树，因此其也由叶子节点和非叶子节点组成。非叶子节点存放的是查询的search key（键值）。其构造包括三个字段：space | marker | offset|

>search key一共占9字节，其中space占4字节，marker占1字节、offset占4字节。space表示待插入记录所在的表空间id，在InnoDB存储引擎中，每个表有一个唯一的space id，可以通过space id查询得知是哪张表。marker是用来兼容老版本的insert buffer。offset表示页所在的偏移量。

>当一个辅助索引需要插入到页（space， offset）时，如果这个页不在缓冲池中，那么InnoDB存储引擎首先根据上述规则构造一个search key，接下来查询insert buffer这棵B+树，然后再将这条记录插入到insert buffer B+树的叶子节点中。

>对于插入到insert buffer B+树叶子节点的记录，需要根据如下规则进行构造：space | marker | offset | metadata | secondary index record|

>启用insert buffer索引后，辅助索引页（space、page_no）中的记录可能被插入到insert buffer B+树中，所以为了保证每次merge insert buffer页必须成功，还需要有一个特殊的页来标记每个辅助索引页（space、page_no）的可用空间。这个页的类型为insert buffer bitmap。

**何时将缓存合并到文件中去**
>* 辅助索引页被读取到缓冲池时；(索引页被查询的时候,相当于多次插入修改删除的合并)
>* insert buffer bitmap页追踪到该辅助索引页已无可用空间时(缓存池不够)
>* master thread(每10S或者每秒)

**insert buffer带来的问题**
>* 可能导致数据库宕机后实例恢复时间变长。如果应用程序执行大量的插入和更新操作，且涉及非唯一的聚集索引，一旦出现宕机，这时就有大量内存中的插入缓冲区数据没有合并至索引页中，导致实例恢复时间会很长。
>* 在写密集的情况下，插入缓冲会占用过多的缓冲池内存（innodb_buffer_pool），默认情况下最大可以占用1/2，这在实际应用中会带来一定的问题。





