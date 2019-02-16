# 一、哈希表概述
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;哈希表是根据关键码值(Key value)而直接进行访问的数据结构。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做散列函数，存放记录的数组叫做散列表或哈希表。具体表现为：存储位置 = f(key);
哈希表的存储过程如下：
1. 根据 key 计算出它的 hash 值 h。
2. 假设箱子的个数为 n，那么这个键值对应该放在第(h%n)个箱子中。
3. 如果该箱子中已经有了键值对，就是用开放寻址法或者拉链法解决冲突。

负载因子：它用来衡量哈希表的 **空/满** 程度，一定程度上也可以体现查询的效率，计算公式为：
```
负载因子 = 总键值对数 / 箱子个数
```
负载因子越大，意味着哈希表越满，越容易导致冲突，性能也就越低。因此，一般来说，当负载因子大于某个常数(可能是1， 或者 0.75等)时，哈希表将自动扩容。

哈希表在自动扩容时，一般会创建两倍于原来个数的箱子，因此即使 key 的哈希值不变，对箱子个数取余的结果也会发生改变，因此所有键值对的存放位置都有可能发生改变，这个过程也称为重哈希(rehash)。

哈希表的扩容并不是总是能够有效解决负载因子过大问题。假设所有 key 的哈希值都一样，那么即使扩容以后他们的位置也不会改变。虽然负载因子会降低，但实际存储在每个箱子中的链表长度并不发生改变，因此也就不能提高哈希表的查询性能。

基于以上总结，我们可以发现哈希表的两个问题：
1. 如果哈希表中本来箱子就比较多，扩容时需要重新哈希并移动数据，性能影响大。
2. 如果哈希函数设计不合理，哈希表在极端情况下会变成线性表，性能极低。

那我们就带着问题去看看 Redis 源码是如何解决以上问题的。接下来我们一步步来分析Redis Dict Reash的机制和过程。
##### (1) Redis 哈希表结构体：
```
/* 
  * hash表结构定义 
  * 这里就是我们的 hash table 结构，存储这数据库中所有的 key-value，
  * 不管是 sds，还是 set，还是什么结构，都存储在里面
  */
typedef struct dictht { 
    dictEntry **table;   // 哈希表数组
    unsigned long size;  // 哈希表的大小
    unsigned long sizemask; // 哈希表大小掩码
    unsigned long used;  // 哈希表现有节点的数量
} dictht; 
```
实体化一下，如下图所指一个大小为4的空哈希表（Redis默认初始化值为4）：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-40d00229412f65fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### (2) Redis 哈希桶
Redis 哈希表中的table数组存放着哈希桶结构（dictEntry），里面就是Redis的键值对；类似Java实现的HashMap，Redis的dictEntry也是通过链表（next指针）方式来解决hash冲突：
```
/* 哈希桶 */
typedef struct dictEntry { 
    void *key;     // 键定义
    // 值定义
    union { 
        void *val;    // 自定义类型
        uint64_t u64; // 无符号整形
        int64_t s64;  // 有符号整形
        double d;     // 浮点型
    } v;     
    struct dictEntry *next;  //指向下一个哈希表节点
} dictEntry;
```

![image.png](https://upload-images.jianshu.io/upload_images/10204326-aeefcd02e676faf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

(3) 字典
Redis Dict 中定义了两张哈希表，是为了后续字典的扩展作Rehash之用：
 ```
/* 字典结构定义 */
typedef struct dict { 
    dictType *type;  // 字典类型
    void *privdata;  // 私有数据
    dictht ht[2];    // 哈希表[两个]
    long rehashidx;   // 记录rehash 进度的标志，值为-1表示rehash未进行
    int iterators;   //  当前正在迭代的迭代器数
} dict;
```
![image.png](https://upload-images.jianshu.io/upload_images/10204326-8ec86898e6a297bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

总结一下：

1. 在Cluster模式下，一个Redis实例对应一个RedisDB(db0);
2. 一个RedisDB对应一个Dict;
3. 一个Dict对应2个Dictht，正常情况只用到ht[0]；ht[1] 在Rehash时使用。

如上，我们回顾了一下Redis KV存储的实现。接下来我们就深入到 Redis 源码里面去看看它的hashtable 有何神奇之处。
#二、操作哈希表
##### (1) 往哈希表里面添加一个元素：dictAdd()
```
/*
 * 往 hash表里面添加一个元素
 */
int dictAdd(dict *d, void *key, void *val) {
    // 先添加一个 key ，这里会判断当前 key 应该存在的位置
    // 还有当前 ht 是否正在扩容
    dictEntry *entry = dictAddRaw(d, key, NULL);

    if (!entry) return DICT_ERR;
    // 设置值
    dictSetVal(d, entry, val);
    return DICT_OK;
}

/*
 * 往 hash表里面添加key
 */
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing) {
    long index;
    dictEntry *entry;
    dictht *ht;
    // 判断 dict 是否正在扩容
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* 获取新元素的索引，如果返回 -1 则说明该元素已经存在 */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d, key), existing)) == -1)
        return NULL;


    // 是否在扩容，如果正在扩容，则往 ht[1] 里面添加元素
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    // 分配内存
    entry = zmalloc(sizeof(*entry));
    // 头插法
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. 设置 redisEntry 的 key */
    dictSetKey(d, entry, key);
    return entry;
}
```
看到 dictAddRaw 方法我们发现里面做了这样几件事情
① 判断 dict 是否正在扩容，如果正在扩容则再尝试步长为1的扩容
② 计算 key 的hash 值，判断该值是否已经存在，如果存在则直接返回
③ 正在扩容，则往 ht[1] 里面添加数据，否则往 ht[0] 里面添加数据
④ 分配内存，每个hash节点维护这一个冲突节点链表，则用头插法插入节点
⑤ set dict key
以上步骤其实就包含了一个key 是如何添加到 dict(字典)中去的。但是里面还有好多细节值得我们去挖掘。为了更加明确目标，接下来抛出几个问题，然后我们再一一解答这些问题。
① 如何计算 key 的 hash 值，而且如何计算该key 在 hashtable 中的 index？
② Redis 是如何解决 key 冲突问题？
③ 每次扩容的大小是多少？扩容的时候有新的命令请求到底去哪个 ht中找数据？
④ 扩容的时如何将 ht[0] 中的数据转移到 ht[1] 中？ht[1] 又是怎么替换 ht[0] 的？
根据吉德林法则：如果能把某个问题清清楚楚地写下来，那这个问题就已经解决一半了。
先看如何计算 key 的hash 值，还有计算 key 在hashtable 中的index
```
/*
  * 计算 key 的hash 值
  * 我们可以看到实际上还是调用字典本身 type 指向的 hash 函数
  */
#define dictHashKey(d, key) (d)->type->hashFunction(key)

/**
 * 真正计算 key 的 hash 值的函数
 * 
 * @param  keyp [description]
 * @return      [description]
 */
unsigned int dictKeyHash(const void *keyp) {
    unsigned long key = (unsigned long)keyp;
    key = dictGenHashFunction(&key,sizeof(key));
    key += ~(key << 15);
    key ^=  (key >> 10);
    key +=  (key << 3);
    key ^=  (key >> 6);
    key += ~(key << 11);
    key ^=  (key >> 16);
    return key;
}
```
如果hash 函数设计的好的话，冲突节点是很少的，redis 里面使用了泊松分布来设计hash 函数的。对泊松分布感兴趣的同学可以百度，哈哈。接下来我们可以看看redis是怎么计算每个 key 在hash表中的下标 index 的。下面我们就来看看index 的计算吧。
```
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing) {
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    for (table = 0; table <= 1; table++) {
        // 计算下标
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        // 如果当前位置已经存在元素
        while (he) {
            // 1. 比较两个元素是否一样
            if (key == he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            // 2. 两个元素不一样，采用头插法解决冲突问题
            he = he->next;
        }
        // 如果当前没有进行 rehash，则不需要查找 ht[1],直接退出即可
        if (!dictIsRehashing(d)) break;
    }
    // 返回 idx
    return idx;
}
```
看到上面计算key 的下标方法，其实我们也就把第二个问题解决了，redis 是采用头插法解决头结点key hash 冲突问题。
接下来我们就来看看每次应该扩容多少合适。我们看到上面的方法里面有一个 _dictExpandIfNeeded(d) 的方法，我们看看里面具体的实现吧。
```
/* 如果需要扩展hash表 */
static int _dictExpandIfNeeded(dict *d) {
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* 
     * 实际扩容触发机制：
     * 如果哈希表的已用节点数 >= 哈希表的大小，并且以下条件任一个为真：
     * 1) dict_can_resize 为真 
     * 2) 已用节点数除以哈希表大小之比大于 dict_force_resize_ratio=5
     * 那么调用 dictExpand 对哈希表进行扩展,扩展的体积至少为已使用节点数的两倍 
     */  
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used / d->ht[0].size > dict_force_resize_ratio)) {
        return dictExpand(d, d->ht[0].used * 2);
    }
    return DICT_OK;
}
```
上面 size * 2 并不是将哈希表扩容成 size 的两倍，继续深入到 dictExpand() 里面看看吧。
```
/*
    @param d    原字典
    @param size     扩容的 size，大于 size 的最小2次幂
 */
int dictExpand(dict *d, unsigned long size) {
    /*
     * 如果正在进行扩容或者元素个数比扩容到指定大小还要大，
     * 则说明这次扩容是不成功的
     */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;
    // 一个新的 hash table
    dictht n;
    // 重新计算新的 hashtable 的容量
    unsigned long realsize = _dictNextPower(size); 

    /* 扩容之后的大小和原来的大小一样的话则说明这次扩容是不成功的 */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* 给新 hashtable 初始化 */
    n.size = realsize;
    n.sizemask = realsize - 1;
    n.table = zcalloc(realsize * sizeof(dictEntry *));
    n.used = 0;

    //如果这是第一次初始化，那么直接就设置成指定大小
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* 准备对第二个进行增量重组 */
    d->ht[1] = n;
    // rehashidx = 0 表示将要准备扩容了
    d->rehashidx = 0;
    return DICT_OK;
}
```
仔细看，真正的扩容算法其实是 _dictNextPower(size)
```
/* hash 表的容量一定是 2 的幂次方 重新计算hashtable 的容量 */
static unsigned long _dictNextPower(unsigned long size) {
    // DICT_HT_INITIAL_SIZE = 4
    unsigned long i = DICT_HT_INITIAL_SIZE;
    // 扩容算法
    // 如果元素比 long 最大值还要大，那么每次扩容 1 个元素
    // 注意了：这里的 size = used * 2 所以这里至少是扩容2倍
    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    while (1) {
        // 这里并不是 d->ht[0].used*2 的两倍，而是大于 d->ht[0].used*2 的最小2次幂
        // _dictExpandIfNeeded() 该方法可以看出 dictExpand(d, d->ht[0].used*2)
        if (i >= size)
            return i;
        i *= 2;
    }
}
```
DICT_HT_INITIAL_SIZE初始化值为4，通过上述表达式，取当4*2^n >= ht[0].used*2的值作为字典扩展的size大小。即为：ht[1].size 的值等于第一个大于等于ht[0].used*2的2^n的数值。不知道这里有没有读者有困惑，为什么一定要是2的n次方呢？主要有两个原因：
###### 1.减小哈希冲突概率 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;假如当前table数组长度为len，插入节点时，需要对key的hashcode进行哈希，然后跟len-1相与(得到的值一定小于len，避免数组越界) 如果len是2的N次方，那么len-1的后N位二进制一定是全1。假设有两个key，他们的hashcode不同，分别为hashcode1和hashcode2 ，hashcode1和hashcode2 分别与一个后N位全1的二进制相与，结果一定也不同 但是，如果hashcode1和hashcode2 分别与一个后N位非全1的二进制相与，结果有可能相同。也就是说，如果len是2^N，不同hashcode的key计算出来的数组下标一定不同； 否则，不同hashcode的key计算出来的数组下标一定相同。所以Redis 长度为全1，可以减小哈希冲突概率。
###### 2.提高计算下标的效率 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果len的二进制后n位非全1，与len-1相与时，0与1相与需要取反。如果len的二进制后n位全1，完全不需要取反。
如果len为2^N，那么与len-1相与，跟取余len等价，而与运算效率高于取余。如果len不是2^N，与len-1相与，跟取余len不等价。

整了这么多代码，读者都看烦了吧，下面总结一下上面的逻辑：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-c23b5f2cfbab156c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;讲到现在我们好像还是不知道 Redis 是怎么将KV从 ht[0] 数组中转移到 ht[1] 数组中去的。不知道现在会不会有人想，不就是将两个数组拷贝一下么，有啥好讲解的。而且Redis 还是将数据放在内存中，那速度肯定刷刷就拷贝完了。其实这样想也是有道理的，但是不要忘了，Redis 中并不是只存了几个KV，有可能存了上千万的KV或者上亿的KV，如果一下子就转移过去，这样肯定会阻塞 Redis 服务器的。那下面我们就看看 Redis 到底是怎么转移的吧。废话少说，上代码
```
/* 
 * 执行增量重新哈希的N个步骤。 如果仍有，则返回1
 * 键从旧的哈希表移到新的哈希表，否则返回0。
 *
 * 请注意，重新调整步骤包括移动一个存储桶（可能有更多存储空间）
 * 然而，从旧的哈希表到新的哈希表中，我们使用链接时只有一个键）
 * 因为散列表的一部分可能由空白组成，所以它不是保证这个函数会重新扫描一个桶，
*  因为它将会在最多N * 10个空桶中进行访问，否则将会访问
 * 它所做的工作将会被解除，并且该功能可能会阻塞很长一段时间。
 * @param  n [就是步长，每次迁移的步长，固定 100]
 */
int dictRehash(dict *d, int n) {
    // 最大访问空桶数量，进一步减小可能引起阻塞的时间。
    int empty_visits = n * 10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    /*
     * 扩容时，每次只移动 n 个元素，防止 redis 阻塞
     */
    while (n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;
        assert(d->ht[0].size > (unsigned long) d->rehashidx);
        // 一旦超出最大空桶的范围则直接退出
        while (d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            // empty_visits 最大空桶数量
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* 将key 转移到新的 ht中去 */
        while (de) {
            uint64_t h;
            nextde = de->next;
            // 将 ht[0] 中的元素迁移到 ht[1] 中去
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        // 通过 rehashidx 参数记录当前转移数据的位置，方便下次转移
        d->rehashidx++;
    }
    /* 检查是否已经 rehash 完毕了 */
    if (d->ht[0].used == 0) {
        // 释放 ht[0]
        zfree(d->ht[0].table);
        // 将 ht[1] 赋值给 ht[0]
        d->ht[0] = d->ht[1];
        // 重置 ht[1]，等待下一次扩容
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }
    /* More to rehash... */
    return 1;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这里只有转移的操作，并没有指明数据是什么时候转移的，触发机制是在哪里。带着这个疑问，我们继续。
```
dict.c:
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    int rehashes = 0;
    // 每次扩容步长为 100，超过了指定时间就退出
    // 源码里面就是直接写死的 100
    while (dictRehash(d, 100)) {
        rehashes += 100;
        if (timeInMilliseconds() - start > ms) break;
    }
    return rehashes;
}

server.c:
int incrementallyRehash(int dbid) {
    /* Keys dictionary */
    if (dictIsRehashing(server.db[dbid].dict)) {
        dictRehashMilliseconds(server.db[dbid].dict,1);
        return 1; /* already used our millisecond for this loop... */
    }
    /* 过期字典表 */
    if (dictIsRehashing(server.db[dbid].expires)) {
        dictRehashMilliseconds(server.db[dbid].expires,1);
        return 1; /* already used our millisecond for this loop... */
    }
    return 0;
}

server.c:
/*
 * 此函数处理我们需要执行的“后台”操作在Redis数据库中递增，例如 key 到期，rehashing
 */
void databasesCron(void) {
        ···

        /* Rehash */
        if (server.activerehashing) {
            for (j = 0; j < dbs_per_call; j++) {
                int work_done = incrementallyRehash(rehash_db);
                if (work_done) {
                    /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
                    break;
                } else {
                    /* 如果当前 db 不需要 rehash，那么将尝试下一个db */
                    rehash_db++;
                    rehash_db %= server.dbnum;
                }
            }
        }
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过上面的代码我们会发现，Redis 是通过执行 databaseCron 任务来执行渐进式的 rehashing的。而不是一蹴而就的。每次转移的步长是 100。下面详细讲一下具体步骤吧。
① 为 ht[1] 分配空间，让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
② 在字典中维持一个索引计数器变量 rehashidx，将它的值设置为0，表示 rehash 工作正式开始。
③ 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]，当 rehash 工作完成之后，程序将 rehashidx 属性的值增一。
④ 随着字典操作不断执行，最终在某个时间点上，ht[0] 的所有键值对都会被 rehash 至 ht[1]，这时 rehashidx 的值会被设置为-1，表示rehash 已经完成。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;渐进式 rehash 的好处在于它采取分而治之的方式，将 rehash 键值对所需的计算工作均匀摊到对字典的每个添加、删除、查找和更新操作上，避免了集中式 rehash 而带来的庞大计算量。这里是从《Redis设计与实现》中借鉴过来的图解，可以帮助大家更加形象的理解 rehash 过程：
准备开始 rehash：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-a86d32e3277db8c0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rehash 索引 0 上的键值对：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-deedd95b18834bb3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rehash 索引1上的键值对：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-eac68b1ff1cbb2e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rehash 索引 2 上的键值对：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-73f52edb6b464877.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rehash 索引 3 上的键值对：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-98d8c682f6bdc7e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

rehash 执行完毕：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-fc0f28b748ac3d53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;文章写到这里 Redis 哈希表的扩容操作讲的也差不多了，当时 Redis 哈希表操作还没有结束，由于篇幅太长，后续会把Redis 的哈希表的查找元素、删除元素、使用 Scan 遍历元素的操作补上。文章中有任何错误或者写作方式有任何可以改进的意见，欢迎大家给我留言。