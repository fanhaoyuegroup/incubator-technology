# 为什么选择跳跃表
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;目前经常使用的平衡数据结构有：B树，红黑树，AVL树，Splay Tree, Treep等。想象一下，给你一张草稿纸，一只笔，一个编辑器，你能立即实现一颗红黑树，或者AVL树出来吗？ 很难吧，这需要时间，要考虑很多细节，要参考一堆算法与数据结构之类的树，还要参考网上的代码，相当麻烦。用跳表吧，跳表是一种随机化的数据结构，目前开源软件 Redis 和 LevelDB 都有用到它，它的效率和红黑树以及 AVL 树不相上下，但跳表的原理相当简单，只要你能熟练操作链表，就能轻松实现一个 SkipList。
# 有序表的搜索
考虑一个有序表
![image.png](https://upload-images.jianshu.io/upload_images/10204326-348fd03281538755.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从该有序表中搜索元素 < 23, 43, 59 > ，需要比较的次数分别为 < 2, 4, 6 >，总共比较的次数
为 2 + 4 + 6 = 12 次。有没有优化的算法吗?  链表是有序的，但不能使用二分查找。类似二叉
搜索树，我们把一些节点提取出来，作为索引。得到如下结构：

![image](http://upload-images.jianshu.io/upload_images/10204326-dee207d775ffa1bd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 这里我们把 < 14, 34, 50, 72 > 提取出来作为一级索引，这样搜索的时候就可以减少比较次数了。
我们还可以再从一级索引提取一些元素出来，作为二级索引，变成如下结构：

  ![image](http://upload-images.jianshu.io/upload_images/10204326-bdcdcb9aec57bd33.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里元素不多，体现不出优势，如果元素足够多，这种索引结构就能体现出优势来了。
# 跳跃表
下面的结构是就是跳表：
 其中 -1 表示 INT_MIN， 链表的最小值，1 表示 INT_MAX，链表的最大值。

![image](http://upload-images.jianshu.io/upload_images/10204326-40ef4fe6ffb397a3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

跳表具有如下性质：
(1) 由很多层结构组成
(2) 每一层都是一个有序的链表
(3) 最底层(Level 1)的链表包含所有元素
(4) 如果一个元素出现在 Level i 的链表中，则它在 Level i 之下的链表也都会出现。
(5) 每个节点包含两个指针，一个指向同一链表中的下一个元素，一个指向下面一层的元素。

## 跳表的搜索

![image](http://upload-images.jianshu.io/upload_images/10204326-98cf6cafa703e4a1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

例子：查找元素 117
(1) 比较 21， 比 21 大，往后面找
(2) 比较 37,   比 37大，比链表最大值小，从 37 的下面一层开始找
(3) 比较 71,  比 71 大，比链表最大值小，从 71 的下面一层开始找
(4) 比较 85， 比 85 大，从后面找
(5) 比较 117， 等于 117， 找到了节点。

## 跳表的插入

先确定该元素要占据的层数 K（采用丢硬币的方式，这完全是随机的）
然后在 Level 1 ... Level K 各个层的链表都插入元素。
例子：插入 119， K = 2

![image](http://upload-images.jianshu.io/upload_images/10204326-49b83bf963c001d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果 K 大于链表的层数，则要添加新的层。
例子：插入 119， K = 4

![image](http://upload-images.jianshu.io/upload_images/10204326-780bb610d31b2b16.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 跳表的高度。
n 个元素的跳表，每个元素插入的时候都要做一次实验，用来决定元素占据的层数 K，跳表的高度等于这 n 次实验中产生的最大 K，待续。。。

## 跳表的空间复杂度分析
根据上面的分析，每个元素的期望高度为 2， 一个大小为 n 的跳表，其节点数目的期望值是 2n。

## 跳表的删除
在各个层中找到包含 x 的节点，使用标准的 delete from list 方法删除该节点。
例子：删除 71

![image](http://upload-images.jianshu.io/upload_images/10204326-143646b547b6ffb0.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
好啦，上面我们跳跃表就介绍完了，接下来我们看看 Redis中是如何实现跳跃表的把。我们知道 Redis 中 zset 有序集合底层就使用了跳跃表来存储数据，那么我们就来看看 zset 结构把。老规矩，我看redis 源码都是从命令入手的，那么我们就来看看 zadd 这个命令做了哪些事情把。
# Redis 跳跃表实现
在讲 Redis 实现的跳跃表之前我们先讲讲 Redis 有序集合的组成成分吧！
```
/**
 * ZSETs use a specialized version of Skiplists
 * 跳跃表中的数据节点
 */
typedef struct zskiplistNode {
    sds ele;
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
    	// 前进指针
        struct zskiplistNode *forward;
        /**
         * 跨度实际上是用来计算元素排名(rank)的，
         * 在查找某个节点的过程中，将沿途访过的所有层的跨度累积起来，
         * 得到的结果就是目标节点在跳跃表中的排位
         */
        unsigned long span;
    } level[];
} zskiplistNode;


/**
 * 跳跃表结构体
 */
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

/**
 * 有序集合结构体
 */
typedef struct zset {
    /*
     * Redis 会将跳跃表中所有的元素和分值组成 
     * key-value 的形式保存在字典中
     * todo：注意：该字典并不是 Redis DB 中的字典，只属于有序集合
     */
    dict *dict;
    /*
     * 底层指向的跳跃表的指针
     */
    zskiplist *zsl;
} zset;
```
## zadd 命令（添加元素）
之所以从 zadd 入手，是因为我们一开始使用 zadd 添加有序集合的时候，该有序集合是不存在的，redis 需要先创建一个有序集合，这样我们才能从源头开始看起，那么我们就来看看 zaddCommand 方法的实现把。
```
/**
 * 往有序集合中添加元素
 * 
 * @param c 客户端数据 
 */
void zaddCommand(client *c) {
    zaddGenericCommand(c, ZADD_NONE);
}
```
由上可知，底层还是调用的 zaddGenericCommand() 这个方法，我们继续往下看
```
void zaddGenericCommand(client *c, int flags) {
    // 先忽略一些干扰代码
    ... 
    /* 从字典中查找该有序集合是否存在，如果不存在则创建 */
    zobj = lookupKeyWrite(c->db, key);
    if (zobj == NULL) {
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        // 如果配置文件中关闭了 ziplist 结构存储，则直接创建用跳跃表存储的有序集合
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx + 1]->ptr)) {
            zobj = createZsetObject();
        } else {
            // 从这里我们可以看出底层有序集合一开始是使用 ziplist 存储元素的，ziplist 下次再说
            zobj = createZsetZiplistObject();
        }
        // 往字典里面保存我们新创建的 有序集合
        dbAdd(c->db, key, zobj);
    } else {
        // 不是有序集合的结构报错
        ...
    }

    for (j = 0; j < elements; j++) {
        double newscore;
        score = scores[j];
        int retflags = flags;

        ele = c->argv[scoreidx + 1 + j * 2]->ptr;
        // 在这里我们终于看到了往 zset 里面添加元素的代码了
        int retval = zsetAdd(zobj, score, ele, &retflags, &newscore);
        ...
    }
    ...
}

```
调用链很长，还是要大家耐着性子继续往下看！
```
int zsetAdd(robj *zobj, double score, sds ele, int *flags, double *newscore) {
     // 忽略干扰代码
     ...
    /* 编码格式是 ziplist */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 往 ziplist压缩列表中添加元素
        ...
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) { // 往跳跃表中添加元素
        zset *zs = zobj->ptr;
        zskiplistNode *znode;
        dictEntry *de;
        // todo: 在字典中查找指定元素的，注意这个字典是 zset 结构体中的字典结构
        de = dictFind(zs->dict, ele);
        if (de != NULL) {
            // 如果开启了 NX 模式，且元素已经在有序集合中存在，则直接返回
            /* NX? Return, same element already exists. */
            if (nx) {
                *flags |= ZADD_NOP;
                return 1;
            }
            // 获取字典中的有序集合中元素的分值
            curscore = *(double *) dictGetVal(de);

            /* 如果需要，准备增量的分数。 */
            if (incr) {
                // 增量分值
                score += curscore;
                if (isnan(score)) {
                    *flags |= ZADD_NAN;
                    return 0;
                }
                if (newscore) *newscore = score;
            }

            /* 分数变化时删除并重新插入。 */
            if (score != curscore) {
                ...
                // 将元素和分值重新插入到跳跃表中
                znode = zslInsert(zs->zsl, score, node->ele);
                // 释放 node 节点，因为我们可以重用 zslInsert 返回的 znode 节点，节省内存空间
                node->ele = NULL;
                zslFreeNode(node);
               // 删除干扰代码
                ...
            }
            return 1;
        } else if (!xx) {// 非 仅在元素已存在时才执行操作
            // 将有序集合中的元素复制一遍
            ele = sdsdup(ele);
            // 直接插入元素
            znode = zslInsert(zs->zsl, score, ele);
            // 将有序集合中的元素和 score 组成 key-value 存储在 zset 结构体中的字典当中
            serverAssert(dictAdd(zs->dict, ele, &znode->score) == DICT_OK);
            ...
        } else {
            ...
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* Never reached. */
}
```
大家耐着性子看到这里有没有点小激动，因为我们马上就要看到跳跃表是如何添加元素了，废话少说，撸代码。
```
/*
 * 往跳跃表中添加一个元素
 */
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    // C 语言数组长度一旦确定就不允许修改
    // 这里和层级 level 中指向的元素相同
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // 等级指针
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level - 1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position 将字符插入到指定位置*/
        rank[i] = i == (zsl->level - 1) ? 0 : rank[i + 1];
        // todo: 先根据分值比较，如果分值都相同的情况下，再比较字符串的长度
        // 我们知道有序集合里面的元素都是有序的，那么肯定就有个排序规则
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 // strcmp() 以二进制的方式进行比较，不会考虑多字节或宽字节字符
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /*
     * 我们假设元素不在内部，因为我们允许重复分数，重新插入相同的元素应该永远不会发生
     * 如果元素在内部，则zslInsert（）的调用者应该在哈希表中进行测试已经在里面或没有。大家可以回到前面看看上面一个方法的逻辑判断
     */

    // 使用幂次定律计算节点的层级
    level = zslRandomLevel();
    // 如果计算出的层级比当前跳跃表的层级更高
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            // 这里将超出 zsl 原来层级的指针都指向新节点 o
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    // 创建一个跳跃表节点
    x = zslCreateNode(level, score, ele);
    for (i = 0; i < level; i++) {
        // 接链
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        /* update span covered by update[i] as x is inserted here  更新范围由update [i]覆盖，因为x插入此处 */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    /* increment span for untouched levels 增加跨度*/
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    /* 更新后退指针 */
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```
代码看到这里，大家再结合上面跳跃表介绍图形结合，理解起来应该更快，如果大家还有什么疑问，可以给我留言，我会在第一时间给大家回复的。
## zrem 命令（删除元素）
看完添加元素的方法，我们再来看看 redis 是如何删除元素的。
```
void zremCommand(client *c) {
    // 有序集合不存在则直接返回
    ...
    // 我们知道 zrem 可以一次性删除多个元素，这里我们看到 redis 是循环删除元素的
    for (j = 2; j < c->argc; j++) {
        // 删除指定元素
        if (zsetDel(zobj, c->argv[j]->ptr)) deleted++;
        // 如果集合中没有元素了，则直接将该有序集合从字典中移除
        if (zsetLength(zobj) == 0) {
            dbDelete(c->db, key);
            keyremoved = 1;
            break;
        }
    }

    // 删除干扰代码
    ...
}
```
和刚刚添加方法一样，又调用了其他的删除方法，继续往下看。
```
int zsetDel(robj *zobj, sds ele) {
    // 如果编码格式是 ziplist 则从 ziplist 中删除元素
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        ...
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {// 编码格式是跳跃表
        zset *zs = zobj->ptr;
        dictEntry *de;
        double score;
        // 在字典中解除该元素节点，但是并没有释放该节点的内存，会在下面的 skiplist 释放
        de = dictUnlink(zs->dict, ele);
        if (de != NULL) {
            /* Get the score in order to delete from the skiplist later. */
            score = *(double *) dictGetVal(de);

            /* 我们知道有序集合中会将 元素和分值构造成 entry 存储在 字典中，所以要释放 entry */
            dictFreeUnlinkedEntry(zs->dict, de);

            /* 在跳跃表中释放元素 */
            int retval = zslDelete(zs->zsl, score, ele, NULL);
            serverAssert(retval);

            if (htNeedsResize(zs->dict)) dictResize(zs->dict);
            return 1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return 0; /* No such element found. */
}
```
代码进行到这里，我们应该也能猜到 zslDelete 方法里面到底做了哪些操作吧！来，我们来猜一猜。
1. 首先要在跳跃表中定位到要删除的元素吧
2. 我们知道该节点每一层都有前驱、后继指针，那么我们删除这个节点的时候，自然也要改变该节点的每一层节点指针的指向啦。
3. 我们知道 redis 跳跃表中还有跨度的概念，该节点没了，那么肯定要改变相关节点的跨度
4. 我们还知道跳跃表是有序的，有个 rank 排名的概念，删除了一个节点，后面的节点排名肯定也要做相应的改变咯。
5. 节点删除了，最后肯定就是释放该节点空间呗。
上面我们分析完了，实际是不是和我们预估的一样呢，我们看看代码吧
```
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 找到要更新的节点
    for (i = zsl->level - 1; i >= 0; i--) {
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) < 0))) {
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* We may have multiple elements with the same score, what we need
     * is to find the element with both the right score and object. */
    x = x->level[0].forward;
    if (x && score == x->score && sdscmp(x->ele, ele) == 0) {
        // 删除 skiplistNode 节点
        zslDeleteNode(zsl, x, update);
        if (!node)
            // 释放节点空间
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0; /* not found */
}
```
我们看看 redis 是如何删除跳跃表中的元素的
```
/*
 * Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank
 * 该方法会修改跳跃表中的跨度
 *
 * zslDelete，zslDeleteByScore和zslDeleteByRank 使用的内部函数
 */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            // 计算跨度
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    while (zsl->level > 1 && zsl->header->level[zsl->level - 1].forward == NULL)
        zsl->level--;
    zsl->length--;
}
```
## zscore 命令（获取元素分值）
我们可以想想获取某个值的分值，在跳跃表中我们是不是要先在跳跃表中找到指定节点然后再获取该节点的分值吗？带着这个疑问我们看看源码吧！
```
/**
 * zscore 获取分值
 * @param c 
 */
void zscoreCommand(client *c) {
    robj *key = c->argv[1];
    robj *zobj;
    double score;
    // 如果在字典中找不到该有序集合或者元素类型不是 OBJ_ZSET，直接返回
    if ((zobj = lookupKeyReadOrReply(c, key, shared.nullbulk)) == NULL ||
        checkType(c, zobj, OBJ_ZSET))
        return;
    // 获取分值
    if (zsetScore(zobj, c->argv[2]->ptr, &score) == C_ERR) {
        addReply(c, shared.nullbulk);
    } else {
        addReplyDouble(c, score);
    }
}
```
从上面我们可以看到，分值的获取实际是通过 zsetScore 方法获取的。
 ```
int zsetScore(robj *zobj, sds member, double *score) {
    if (!zobj || !member) return C_ERR;
    // 如果编码格式压缩链表
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        if (zzlFind(zobj->ptr, member, score) == NULL) return C_ERR;
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {// 编码格式是跳跃表
        zset *zs = zobj->ptr;
        // 直接从 zset 中的字典获取指定的 dictEntry
        dictEntry *de = dictFind(zs->dict, member);
        if (de == NULL) return C_ERR;
        // 从 dictEntry 中获取 score
        *score = *(double *) dictGetVal(de);
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return C_OK;
}
```
## zrank 命令（获取某个元素的排名）
当我们想获取有序集合中某个元素的排名时，zrank 命令是我们很好的选择，zrank 命令返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值递增(从小到大)顺序排列。排名以 0 为底，也就是说， score 值最小的成员排名为 0 。下面我们看看他里面具体实现吧！
```
long zsetRank(robj *zobj, sds ele, int reverse) {
    unsigned long llen;
    unsigned long rank;

    llen = zsetLength(zobj);

    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        // 压缩链表 rank 计算方式
        ...
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;
        // 从 zset 中的字典中直接拿到指定元素 dictEntry
        de = dictFind(zs->dict, ele);
        if (de != NULL) {
            score = *(double *) dictGetVal(de);
            rank = zslGetRank(zsl, score, ele);
            /* Existing elements always have a rank. */
            serverAssert(rank != 0);
            if (reverse)
                return llen - rank;
            else
                return rank - 1;
        } else {
            return -1;
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}
```
接下来就是真正 rank 计算的方法咯，不要分心，专心看。
```
unsigned long zslGetRank(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    // 这里从最上层开始查找节点元素的排名
    for (i = zsl->level - 1; i >= 0; i--) {
        while (x->level[i].forward &&
               (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 sdscmp(x->level[i].forward->ele, ele) <= 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        /* x might be equal to zsl->header, so test if obj is non-NULL */
        if (x->ele && sdscmp(x->ele, ele) == 0) {
            return rank;
        }
    }
    return 0;
}
```
好啦，到这里 Redis 中实现的跳跃表大概讲完了，当然还有其他 api 没有讲啦，感兴趣的同学可以自己琢磨琢磨，欢迎给我留言。



