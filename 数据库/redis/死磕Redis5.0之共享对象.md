&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在 Redis 中，内存是很宝贵的资源，我们知道 Redis 之所以快，和它所有数据都在内存中是密不可分的。而内存又是很宝贵的资源，那么 Redis 在使用的内存的时候有没有做什么优化呢？我们一起来探究一下吧。
# redisObject 对象
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了直观的看到 Redis 存储我们设置的值，我们将从 Redis 的网络模块还是讲起，我们知道 Redis 会将我们设置的值保存在输入缓冲区中，那么我们就来看看输入缓冲区 Redis 做了哪些操作吧。
```
/**
 * 从输入缓冲区中读取数据
 */
int processInlineBuffer(client *c) {
    
    ...

    /*
     * Create redis objects for all arguments.
     *
     * todo: 为 client 中所有的参数都创建成一个 redis object 对象
     */
    for (c->argc = 0, j = 0; j < argc; j++) {
        if (sdslen(argv[j])) {
            // 创建一个 object 对象
            c->argv[c->argc] = createObject(OBJ_STRING,argv[j]);
            c->argc++;
        } else {
            sdsfree(argv[j]);
        }
    }
    
    ...
    
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;从 Redis 的源码我们可以知道，Redis存储的所有值对象在内部定义为redisObject 结构体，具体内部结构体：
```
/**
 * Redis 存储的 value 数据都是用 redisObject 来封装的
 * 包括 string，hash，list，set，zset  在内的所有数据类型
 */
typedef struct redisObject {
    /**
     * 表示当前对象使用的数据类型，
     * Redis主要支持5种数据类型:string,hash,list,set,zset。
     * 可以使用type {key}命令查看对象所属类型，
     * type命令返回的是值对象类型，键都是string类型。
     */
    unsigned type:4;
    /**
     * 表示Redis内部编码类型，encoding在Redis内部使用，
     * 代表当前对象内部采用哪种数据结构实现。
     * 理解Redis内部编码方式对于优化内存非常重要 ，
     * 同一个对象采用不同的编码实现内存占用存在明显差异，
     * 具体细节见之后编码优化部分。
     */
    unsigned encoding:4;
    /**
     * 记录对象最后一次被访问的时间，当配置了
     * maxmemory 和 maxmemory-policy=volatile-lru | allkeys-lru 时，
     * 用于辅助LRU算法删除键数据。
     * 可以使用 object idletime {key} 命令在不更新 lru 字段情况下查看当前键的空闲时间。
     */
    unsigned lru:LRU_BITS; 
    /**
     * 记录当前对象被引用的次数，用于通过引用次数回收内存，
     * 当refcount=0时，可以安全回收当前对象空间。
     * 使用 object refcount {key} 获取当前对象引用。
     */
    int refcount;
    /**
     * 与对象的数据内容相关，如果是整数直接存储数据，否则表示指向数据的指针。
     * Redis在3.0 之后对值对象是字符串且长度 <=39 字节的数据，
     * 内部编码为 embstr 类型，字符串 sds 和 redisObject 一起分配，
     * 从而只要一次内存操作。
     * todo: 因此在高并发的场景尽量是我们的字符串保持 39 字节内，
     * 减少创建redisObject内存分配次数从而提高性能。
     */
    void *ptr;			// 指向底层实现数据结构的指针
} robj;

```
具体结构内部图如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/10204326-f3c92a1b6de9e8ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过上面我们会知道，Redis 会把我们的 value 都用一个 redisObject 对象存储。创建一个 redisObject 对象至少需要 16 个字节。（如果知道 16 个字节是怎么来的大家可以百度一下 c 语言中各个类型所占字节数，这里我只告诉你 type + encoding 占一个字节，lru 占 3 个字节）如果我们总是设置相同的 value，这样 redisObject 就会成倍增长，这样是不是有点浪费内存呢？我们是不是可以把这些对象都共享起来呢？带着这个疑问我们继续往下探索吧。
# Redis 共享对象
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;很多人都知道 Redis 内部维护[0-9999]的整数对象池。创建大量的整数类型redisObject 存在内存开销，每个redisObject内部结构至少占16字节，甚至超过了整数自身空间消耗。所以Redis内存维护一个[0-9999]的整数对象池，用于节约内存。 除了整数值对象，其他类型如list,hash,set,zset内部元素也可以使用整数对象池。因此开发中在满足需求的前提下，尽量使用整数对象以节省内存。整数对象池在 Redis 中通过变量 REDIS_SHARED_INTEGERS 定义，不能通过配置修改。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;然而很多人好像也只知道 Redis 会将 [0-9999]的整数对象共享起来，那么除了这些整数之外，Redis 还会创建其他共享对象么？答案是肯定。下面我们就来看看 Redis 到底还维护了哪些对象吧。
```
/**
 * todo: Redis 共享变量
 * 共享对象结构体，注意里面每一个共享对象都是 robj(redisObject) 对象
 * 
 * 这里面有部分值是要放到输出缓冲区里面的，为了保证内存中只有一份值，所以
 * 可以将这些对象共享起来，这样可以节约内存。
 */
struct sharedObjectsStruct {
    robj *crlf, *ok, *err, *emptybulk, *czero, *cone, *cnegone, *pong, *space,
    *colon, *nullbulk, *nullmultibulk, *queued,
    *emptymultibulk, *wrongtypeerr, *nokeyerr, *syntaxerr, *sameobjecterr,
    *outofrangeerr, *noscripterr, *loadingerr, *slowscripterr, *bgsaveerr,
    *masterdownerr, *roslaveerr, *execaborterr, *noautherr, *noreplicaserr,
    *busykeyerr, *oomerr, *plus, *messagebulk, *pmessagebulk, *subscribebulk,
    *unsubscribebulk, *psubscribebulk, *punsubscribebulk, *del, *unlink,
    *rpop, *lpop, *lpush, *zpopmin, *zpopmax, *emptyscan,
    *select[PROTO_SHARED_SELECT_CMDS],
    // todo: 存了 [0, OBJ_SHARED_INTEGERS) 的数字常量
    *integers[OBJ_SHARED_INTEGERS],
    *mbulkhdr[OBJ_SHARED_BULKHDR_LEN], /* "*<value>\r\n" */
    *bulkhdr[OBJ_SHARED_BULKHDR_LEN];  /* "$<value>\r\n" */
    sds minstring, maxstring;
};
```
Redis 将所有维护的共享对象都放在 sharedObjectsStruct 结构体中，接下来看看 Redis 是怎么给这些共享对象赋值的吧。
```
**
 * Redis 共享变量赋值
 */
void createSharedObjects(void) {
    ...
    
    // 这里的值都是要放到 Redis 输出缓冲区里面的，要返回给客户端的
    // 所以都是按照 Redis 协议来赋值的
    shared.ok = createObject(OBJ_STRING, sdsnew("+OK\r\n"));
    shared.err = createObject(OBJ_STRING, sdsnew("-ERR\r\n"));
    
    ...
    
    // 客户端发送 ping 命令时，服务端会发送 pong 命令
    shared.pong = createObject(OBJ_STRING, sdsnew("+PONG\r\n"));
    shared.queued = createObject(OBJ_STRING, sdsnew("+QUEUED\r\n"));
    
    ...
    
    // 这里就是数字常量 [0, OBJ_SHARED_INTEGERS)
    for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
        shared.integers[j] =
                makeObjectShared(createObject(OBJ_STRING, (void *) (long) j));
        shared.integers[j]->encoding = OBJ_ENCODING_INT;
    }
    
   ...
}

/**
 * todo: 这里是对 redis 服务器进行初始化
 *
 * [initServer description]
 */
void initServer(void) {
    
    ...
    
    // todo: 创建一些共享对象
    // Redis 在初始化的时候就会给自己维护的共享对象赋值
    createSharedObjects();
   
    ...
}

```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;上面方法就是给 Redis 维护的所有共享对象赋值，我并没有把所有的共享对象都列出来，如果大家感兴趣可以看看 Redis 源码里面，找到该方法，仔细研究研究。上面我只列出来一些常见的共享对象，看到上面大家应该会很熟悉，因为我们看到了 OK、-ERR、QUEUED 这些常见字符串，大家仔细思考就知道，这些值都是我们设置命令 Redis 给我们返回的响应。是的，Redis 会把一些常见的给客户端回复的字符串共享起来，以此来节省内存。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;讲到这里大家肯定就会有疑问，我们会往 Redis 里面存储很多字符串，这些字符串大多数都是重复的，那么我们把这戏字符串都设置成共享对象，岂不是会节省更多的内存空间？真的是这样吗？思考一下吧（答案：Redis 不会共享包含字符串的对象）
# Why Redis 不共享包含字符串的对象？
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当服务器考虑将一个共享对象设置为键的值对象时，程序需要先检查给定的共享对象和键创建的目标对象是否完全相同，只有在共享对象和目标对象完全相同的情况下，程序才会将共享对象用作键的值对象，而一个共享对象保存的值越复杂，验证共享对象和目标对象是否相同所需的复杂度就会越高，消耗的 CPU 时间也会越多：
1. 如果共享对象是保存整数值(0~9999)的字符串对象，那么验证操作的复杂度为O(1)
2. 如果共享对象是保存字符串值的字符串对象，那么验证操作的复杂度为 O(N)
3. 如果共享对象是包含了多个值(或者对象) 对象，比如列表对象或哈希对象，那么验证操作的复杂度为 O(N^2)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;因此，尽管共享更复杂的对象可以节约更多的内存，但受到 CPU 时间的限制，Redis 只对包含整数值的字符串对象进行共享。
