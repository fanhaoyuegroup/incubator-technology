# Redis 事务简介
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务功能。事务提供了一种将多个命令请求打包，然后一次性、按顺序的执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求，它会将事务中的所有命令都执行完毕，然后才去处理其他客户端的命令请求。一个事务从开始到结束通常会经历以下三个阶段：
1. 事务开始
2. 命令入队
3. 事务执行
文章也会重点围绕上面三个步骤来讲的。
# 事务组成
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis要想了解 Redis 中的事务自然要了解 Redis 是如何实现事务的，那么我们肯定也就要知道 Redis 用了那些结构来存储我们的事务的，下面我们就来看看事务的组成。
```
/*
 * todo: client 结构体
 *
 * With multiplexing we need to take per-client state.
 * Clients are taken in a linked list.
 */
typedef struct client {
    ...
    // 标记，如果有事务进来 flags |= CLIENT_MULTI 追加事务状态
    int flags;              
    
    ...
    // 事务结构体
    multiState mstate;      /* MULTI/EXEC state */

    ...
    
    // 使用 watch 监控的所有 key
    list *watched_keys; 
    
    ...
} client;
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis Redis 会为每个客户端创建一个 client 结构体，client 结构体里会存储当前客户端的一些信息，而我们的事务信息也会保存在里面，下面我们详细讲解和事务相关的几个字段。
1. flags ：字段采用位运算记录很多状态，当我们标记事务状态的时候只需要将 flags |= CLIENT_MULTI 即可追加事务状态。
2. mstate ：事务状态结构体，里面会存储我们的命令列表
3. watched_keys ：看名字我们就知道，这个肯定是用监控事务中 key 变化的列表
现在我们来看看 multiState 这个结构体里面到底存储了哪些东西吧。
```
redis> multi
ok

redis> set "name" "practical common lisp"
queued

redis> get "name"
queue

redis> set "author" "peter seibel"
queued

redis> get "author"
queued
```
上面的命令服务器将为客户端创建下图所示的事务状态：
![image.png](https://upload-images.jianshu.io/upload_images/10204326-cc225ee77187d6e0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



```
/**
 * 事务状态结构体
 */
typedef struct multiState {
    /**
     * 事务中命令列表
     */
    multiCmd *commands;     
    /**
     * 事务队列里面命令的个数
     */
    int count;              
    /**
     * 用于同步复制
     */
    int minreplicas;        
    /**
     * 同步复制超时时间
     */
    time_t minreplicas_timeout; 
} multiState;
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 通过上面我们可以看到，multiState 保存了我们事务中所有的命令列表，一旦我们发送提交事务的命令，那么 Redis 就会从 multiState 拿到事务中所有的命令，然后依次执行。上面 commands 保存的是 multiCmd 结构体，而这个结构体里面就保存了我们命令要执行的一些信息。
```
/**
  * 客户端事务命令结构体
  * Client MULTI/EXEC state 
  */
typedef struct multiCmd {
    /**
     * 命令执行的参数列表
     */
    robj **argv;
    /**
     * 命令执行的参数的个数
     */
    int argc;
    /**
     * 具体要执行的命令指针
     */
    struct redisCommand *cmd;
} multiCmd;
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 为了更好的理解上面的结构，我们可以添加几条命令，看看 Redis 到底是如何使用这几个结构体来存储我们的事务命令的。
# 事务开始
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 事务是从 multi 命令开始的，那么我们看看输入 multi 命令，Redis 到底做了哪些操作。我们知道一个 multi 命令在 Redis 里面就对应了一个 multiCommand 方法，那么我就找到该方法一探究竟吧。
```
/**
 * 开启一个事务的命令
 */
void multiCommand(client *c) {
    // 事务不支持嵌套(不支持事务里面再包含事务)
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    // 将客户端的 flags 标志添加一个事务标志
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 从上面我们可以看出，我们使用 multi 开启一个事务的时候，Redis 只是将当前 client 的 flags 追加一个事务标志。如果当前客户端已经开启了事务，那么在当前事务没有结束之前是不允许再发送 multi 命令的。
# 事务入队
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 从上面我们已经知道 multi 命令后 Redis 是如何开启一个事务的，也许现在很多人又会会很好奇，为什么我们输入一个 multi 命令后，redis 就会把 multi 之后的命令都加入命令队列里面呢，下面我就来揭晓这个答案吧。我们来看一下所有 redis 命令的入口吧。
```
int processCommand(client *c) {
    
    ...

    /* Exec the command 这里就是事务命令执行的地方 */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand) {
        queueMultiCommand(c);
        addReply(c, shared.queued);
    } else {
        // todo: call 调用，这里面就会调用非事务命令的方法
        call(c, CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 通过 processCommand 方法我们可以知道，当 client 状态 flags 为 CLIENT_MULTI 事务状态的时候，并且，客户端输入的命令非 exec、discard、multi、watch 命令的时候，Redis 会将输入的命令通过 queueMultiCommand 方法加入事务队列，然后向客户端返回 shared.queued("QUEUED") 字符串。如果不是事物状态，那么 Redis 会马上执行我们输入的命令，看到这里就知道为什么 multi 之后的命令都会加入命令队列了吧。看到这里是否有意犹未尽之意，我们继续往下看，Redis 的 queueMultiCommand 方法具体是如何实现的。
```
/**
 * 将新命令添加到MULTI命令队列中
 * 阅读该方法一定要有 C 语言基础，能看懂指针地址的赋值操作
 */
void queueMultiCommand(client *c) {
    // 事务命令指针，里面会指向真正要执行的命令
    multiCmd *mc;
    int j;
    // 给新增的 multiCmd 计算内存起始地址，在 commands 链表中加入新事务命令
    c->mstate.commands = zrealloc(c->mstate.commands,
                                  sizeof(multiCmd) * (c->mstate.count + 1));
    // 地址赋值，实际就是将 multiCmd 加入 mstate.commands 队尾
    mc = c->mstate.commands + c->mstate.count;
    // 将客户端的命令和参数赋值给事务命令结构体，方便后面执行
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj *) * c->argc);
    memcpy(mc->argv, c->argv, sizeof(robj *) * c->argc);
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 就如上面这样，Redis 会把 multi 之后的命令构造成一个 multiCmd 结构添加到 
mstate.commands 链表后面，方便后续执行。代码看的不过瘾，我们还可以看看执行的流程图，方便大家理解。下图就是上面代码执行的流程图。
![image.png](https://upload-images.jianshu.io/upload_images/10204326-a6370d9559719ebb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 事务执行
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 在事务执行之前，我们不得不提一下 watch 命令。Redis 的官方文档上说，WATCH 命令是为了让 Redis 拥有 check-and-set(CAS) 的特性。CAS 的意思是，一个客户端在修改某个值之前，要检测它是否更改；如果没有更改，修改操作才能成功。通过上面的 client 结构体我们可以知道 watch 监控的 key 是以链表的形式存储在 Redis 的 client 结构体中。具体如下图所示：
![image.png](https://upload-images.jianshu.io/upload_images/10204326-85bcb4e066194449.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

监视键值的过程：
```
/**
 * 这个就是 watch 命令执行步骤
 */
void watchCommand(client *c) {
    int j;

    // 该命令只能出现在 multi 命令之前
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c, "WATCH inside MULTI is not allowed");
        return;
    }
    // 监控指定的 key
    for (j = 1; j < c->argc; j++)
        // todo: 实际监控 key 的操作 
        watchForKey(c, c->argv[j]);
    // 像客户端缓冲区返回 ok
    addReply(c, shared.ok);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 有一些前置操作，比如检测 watch 命令是否在 multi 命令之前，如果不是则直接报错，实际监控 key 的还是 watchForKey 方法，下面我们重点讲解该方法。
```
/* 
 * Watch for the specified key 
 *
 * 监控指定的 key
 */
void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    /* Check if we are already watching for this key */
    listRewind(c->watched_keys, &li);
    while ((ln = listNext(&li))) {
        wk = listNodeValue(ln);
        // 条件满足说明该 key 已经被 watched 了
        if (wk->db == c->db && equalStringObjects(key, wk->key))
            return; /* Key already watched */
    }
    /* This key is not already watched in this DB. Let's add it */
    // 此DB中尚未监视此 key 。 我们加上吧
    // 先从 c->db->watched_keys 中取出该 key 对应的客户端 client
    clients = dictFetchValue(c->db->watched_keys, key);
    // 如果该 client 为 null，则说明该key 没有被 client 监控
    // 则需要在该key 后面创建一个 client list 列表，用来保存
    // 监控了该key 的客户端 client
    if (!clients) {
        clients = listCreate();
        dictAdd(c->db->watched_keys, key, clients);
        incrRefCount(key);
    }
    // 尾插法。将客户端添加到链表尾部,实际服务端也会保存一份
    listAddNodeTail(clients, c);
    /* Add the new key to the list of keys watched by this client */
    // 将新 key 添加到此客户端 watched（监控） 的 key 列表中
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    // 把 wk 赋值给指定的 client 的监控key的结构体中
    listAddNodeTail(c->watched_keys, wk);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 当客户端键值被修改的时候，监视该键值的所有客户端都会被标记为 REDISDIRTY-CAS，表示此该键值对被修改过，因此如果这个客户端已经进入到事务状态，它命令队列中的命令是不会被执行的。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis touchWatchedKey() 是标记某键值被修改的函数，它一般不被 signalModifyKey() 函数包装。下面是 touchWatchedKey() 的实现。
```
// 标记键值对的客户端为REDIS_DIRTY_CAS，表示其所监视的数据已经被修改过
/* "Touch" a key, so that if this key is being WATCHed by some client the
* next EXEC will fail. */
void touchWatchedKey(redisDb *db, robj *key) {
    list *clients;
    listIter li;
    listNode *ln;
    // 获取监视key 的所有客户端
    if (dictSize(db->watched_keys) == 0) return;
        clients = dictFetchValue(db->watched_keys, key);
    if (!clients) return;
        // 标记监视key 的所有客户端REDIS_DIRTY_CAS
        /* Mark all the clients watching this key as REDIS_DIRTY_CAS */
        /* Check if we are already watching for this key */
        listRewind(clients,&li);
    while((ln = listNext(&li))) {
        redisClient *c = listNodeValue(ln);
        // REDIS_DIRTY_CAS 更改的时候会设置此标记
        c->flags |= REDIS_DIRTY_CAS;
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 当用户发出 EXEC 的时候，在它 MULTI 命令之后提交的所有命令都会被执行。从代码的实现来看，如果客户端监视的数据被修改，它会被标记 REDIS_DIRTY_CAS，会调用 discardTransaction() 从而取消该事务。特别的，用户开启一个事务后会提交多个命令，如果命令在入队过程中出现错误，譬如提交的命令本身不存在，参数错误和内存超额等，都会导致客户端被标记 REDIS_DIRTY_EXEC，被标记 REDIS_DIRTY_EXEC 会导致事务被取消。

因此总结一下：

1. REDIS_DIRTY_CAS 更改的时候会设置此标记
2. REDIS_DIRTY_EXEC 命令入队时出现错误，此标记会导致 EXEC 命令执行失败
下面是执行事务的过程：
```
/**
 * 执行事务的命令
 */
void execCommand(client *c) {
    
    ...
    
    // 是否需要将MULTI/EXEC命令传播到slave节点/AOF
    int must_propagate = 0; 
    int was_master = server.masterhost == NULL;

    // 事务有可能会被取消
    if (!(c->flags & CLIENT_MULTI)) {
        // 没事事务可以执行
        addReplyError(c, "EXEC without MULTI");
        return;
    }

    /* 
     * 停止执行事务命令的情况：：
     * 1. 有一些被监控的 key 被修改了
     * 2. 由于命令队列里面出现了错误
     *
     * 第一种情况下失败的EXEC返回一个多块nil对象
     * 技术上它不是错误，而是特殊行为，而在第二个中返回EXECABORT错误
     */
    if (c->flags & (CLIENT_DIRTY_CAS | CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                    shared.nullmultibulk);
        discardTransaction(c);
        goto handle_monitor;
    }

    /* 执行所有排队的命令 */

    // 取消对所有key的监控，否则会浪费CPU资源
    //  因为 redis 是单线程。所以不用担心 key 再被修改了
    unwatchAllKeys(c); 
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyMultiBulkLen(c, c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        // todo: 遇到包含写操作的命令需要将MULTI 命令写入AOF 文件
        if (!must_propagate && !(c->cmd->flags & (CMD_READONLY | CMD_ADMIN))) {
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }
        // 执行我们事务队列里面的命令
        call(c, CMD_CALL_FULL);

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    // 清除事务状态
    discardTransaction(c);
    
    ...

    handle_monitor:
  
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c, server.monitors, c->db->id, c->argv, c->argc);
}

```
如上所说，被监视的键值被修改或者命令入队出错都会导致事务被取消：
```
/**
 * 取消事务，比如遇到事务中的语法错误问题
 */
void discardTransaction(client *c) {
    // 清空命令队列
    freeClientMultiState(c);
    // 初始化命令队列
    initClientMultiState(c);
    // 取消标记flag
    c->flags &= ~(CLIENT_MULTI | CLIENT_DIRTY_CAS | CLIENT_DIRTY_EXEC);
    // 取消对 client 所有被监控的 key
    unwatchAllKeys(c);
}

```
# Redis 事务番外篇
你可能已经注意到「事务」这个词。在学习数据库原理的时候有提到过事务的 ACID，即原子性、一致性、隔离性、持久性。接下来，看看 Redis 事务是否支持 ACID。

1. 原子性，即一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。Redis 事务不支持原子性，最明显的是 Redis 不支持回滚操作。一致性，在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这一点，Redis 事务能够保证。

2. 隔离性，当两个或者多个事务并发访问（此处访问指查询和修改的操作）数据库的同一数据时所表现出的相互关系。Redis 不存在多个事务的问题，因为 Redis 是单进程单线程的工作模式。

3. 持久性，在事务完成以后，该事务对数据库所作的更改便持久地保存在数据库之中，并且是完全的。Redis 提供两种持久化的方式，即 RDB 和 AOF。RDB 持久化只备份当前内存中的数据集，事务执行完毕时，其数据还在内存中，并未立即写入到磁盘，所以 RDB 持久化不能保证 Redis 事务的持久性。再来讨论 AOF 持久化,Redis AOF 有后台执行和边服务边备份两种方式。后台执行和 RDB 持久化类似，只能保存当前内存中的数据集；边备份边服务的方式中，因为 Redis 只是每间隔 2s 才进行一次备份，因此它的持久性也是不完整的！

4. 一致性：待补充

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;还有一个亮点，就是 check-and-set CAS。一个修改操作不断的判断X 值是否已经被修改，直到 X 值没有被其他操作修改，才设置新的值。Redis 借助 WATCH/MULTI 命令来实现 CAS 操作的。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;实际操作中，多个线程尝试修改一个全局变量，通常我们会用锁，从读取这个变量的时候就开始锁住这个资源从而阻挡其他线程的修改，修改完毕后才释放锁，这是悲观锁的做法。相对应的有一种乐观锁，乐观锁假定其他用户企图修改你正在修改的对象的概率很小，直到提交变更的时候才加锁，读取和修改的情况都不加锁。一般情况下，不同客户端会访问修改不同的键值对，因此一般 check 一次就可以 set 了，而不需要重复 check 多次。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;注意：Redis 是不支持事务回滚的，据说作者认为回滚操作会影响 Redis 的性能，所以没有事务回滚的功能。