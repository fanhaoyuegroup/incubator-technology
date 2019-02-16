&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;最近和一些朋友讨论Redis的订阅和发布功能，发现有些公司喜欢用Redis的订阅和发布功能来当作消息中间件来使用，当时我就纳闷，消息中间件比较牛逼的不就是那几个RocketMQ、Kafka、Rabbit MQ等专门的消息中间件么，Redis 的订阅发布功能也能当消息中间件用？带着这个疑问我们一起来探究一下Redis的订阅和发布的实现吧。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;文章分为以下几个部分讲解：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1. 涉及的命令
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. 数据结构
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. 订阅和发布主流程源码分析
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4. Redis 订阅发布功能整的适合做消息中间件吗？
# 一、涉及的命令
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 订阅和发布非常简单，一共就六个命令：psubscribe、publish、pubsub、punsubscribe、subscribe、unsubscribe。具体命令的使用大家可以参考 [黄健宏老师总结的 Redis命令参考](http://redisdoc.com/index.html)，黄健宏老师是我非常崇拜的一个人。黄健宏老师把 redis 所用到的命令都总结好了，我就不在这里再总结一遍了。
# 二、数据结构
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis 订阅和发布有两种类型，一种是频道，还有一种就是模式。我们先看频道的数据结构。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Redis将所有频道的订阅关系都保存在服务器状态的 pubsub_channels 字典里面，这个字典的键是被某个订阅的频道，而键的值是一个链表，链表里面纪录了所有订阅这个频道的客户端：
```
// redisServer 中是使用字典保存的，这里保存着全部的频道
struct redisServer {
    // ...
    // 保存所有频道的订阅关系
    dict *pubsub_channels;
    // ...
}
// client 中也会保存自己感兴趣的频道
typedef struct client {
    // client 中的感兴趣的频道
    dict *pubsub_channels;  
} client;

/*
 * 下面通过 pubsub.c 文件中的 pubsubSubscribeChannel 方法
 * 看看 channel 和 client 具体是如何映射的。
 */

/*
 * 将客户订阅到频道。 如果操作成功，则返回1，如果客户端已订阅该频道，则为0。
 */
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* 查看 client 是否已经订阅了该频道 */
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* 将客户端添加到 channel - >client list 哈希表中 */

        /*
         * 查找指定频道是否在 pubsub_channels 字典中存在，
         * 如果存在直接将客户端添加到 clients 尾部即可。
         * 否则创建一个 clients 链表，然后将 client 添加到 clients 中
         */
        de = dictFind(server.pubsub_channels,channel);
        // 如果根据该 channel 查出的值为 null，说明字典中还没有该频道信息
        if (de == NULL) {
            // 从这里我们可以看出多个客户端是通过链表连接在一起的
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            clients = dictGetVal(de);
        }
        // 频道已经存在，直接添加到尾部
        listAddNodeTail(clients,c);
    }
    ...
}

```
通过源码我们脑海中应该有个大概的印象了，接着我们举个栗子加深印象。比如：
① client-1、client-2、client-3 三个客户端正在订阅 "order.it" 频道
② client-4 正在订阅 "order.sport" 频道
③ client-5 和 client-6 两个客户端正在订阅 "order.business" 频道
则结构如下图：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-7e8118003ffe81b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面就是频道的订阅关系图，模式和频道类似，都是存储到服务器状态中，但是具体的数据结构却大不相同。
```
struct redisServer {
    // ...
    // 保存所有模式的订阅关系
    list *pubsub_patterns;
    // ...
}
// client 中也会保存自己感兴趣的模式
typedef struct client {
    // client 中的感兴趣的模式
    list *pubsub_patterns;
} client;

/*
 * 我们可以看到 redisServer 中直接就是使用链表来存储模式的
 * 下面我们看看具体的模式和 客户端的映射关系吧
 */
/**
 * 订阅模式的结构体
 * 也就是 pubsub_patterns 链表中保存的结构
 */
typedef struct pubsubPattern {
    /**
     * 客户端
     */
    client *client;
    /**
     * 模式
     */
    robj *pattern;
} pubsubPattern;


/*
 * 下面我们看看 Redis 是如何构造 pubsubPattern 并添加到 pubsub_patterns 中
 * 通过 pubsub.c 中的 pubsubSubscribePattern 方法我们可以看到全过程
 */

int pubsubSubscribePattern(client *c, robj *pattern) {
    int retval = 0;
    // 查看 client 自己是否已经订阅该模式
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
        // 没有订阅则将 pubsubPattern 结构体加到 client 的 pubsub_patterns 中
        listAddNodeTail(c->pubsub_patterns,pattern);
        incrRefCount(pattern);
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        // 将该模式和订阅该模式的client 添加到服务端的 pubsub_patterns 链表中
        listAddNodeTail(server.pubsub_patterns,pat);
    }
    ...
}
```
举个 demo，比如：
① client-7 正在订阅 "music.*"。
② client-8 正在订阅 "book.*"。
③ client-9 正在订阅 "order.*".
则结构图如下

![image.png](https://upload-images.jianshu.io/upload_images/10204326-39af6fa3440d64ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到这里Redis 的频道和模式的数据结构就解剖完了，同学们都理解了么？看完频道和模式的数据结构，不知道同学们有没有这样的疑问，频道和模式到底有啥区别呢？下面我们就来看看他们之间到底有什么区别。我们还是通过 demo来了解吧。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;现在我们有 client-1、client-2、client-3、client-4 个客户端，我们让 client-1 订阅"order.create"频道，让 client-2 订阅 "order.waitpay"，让 client-3 订阅 "order.pay" 频道，让 client-4 订阅 "order.*" 模式。然后我们分别往 "order.create"、"order.waitpay"、"order.pay" 发送消息，我们看看每个客户端有何变化。 
client-1 订阅 order.create 频道：subscribe order.create
![image.png](https://upload-images.jianshu.io/upload_images/10204326-a8b95b9a1ef56df8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-2 订阅 order.waitpay 频道：subscribe order.waitpay

![image.png](https://upload-images.jianshu.io/upload_images/10204326-7790150c5576cd36.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-3 订阅 order.pay 频道：subscribe order.pay

![image.png](https://upload-images.jianshu.io/upload_images/10204326-c17a80d60c89066c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-4 订阅 order.* 模式：psubscribe order.*

![image.png](https://upload-images.jianshu.io/upload_images/10204326-0f469769c69da943.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们使用一个客户端分别往这几个客户端发送消息：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-8821e966a6dda47a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们看看每个客户端之间的变化
client-1:

![image.png](https://upload-images.jianshu.io/upload_images/10204326-e66f7c9b99971021.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-2:

![image.png](https://upload-images.jianshu.io/upload_images/10204326-24020e16af315e58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-3:

![image.png](https://upload-images.jianshu.io/upload_images/10204326-136c5bef7ac406b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

client-4:

![image.png](https://upload-images.jianshu.io/upload_images/10204326-90bb05f6799350dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们看到client-1、client-2、client-3都只接受了和自己频道相关的消息，但是 client-4 把发向 client-1、client-2、client-3 的消息都接收了，现在大家应该明白了吧，模式其实就是模式匹配的概念，order.* 就表示匹配所有和 order 相关的消息。
# 三、订阅和发布的源码分析
我们就拿 publish order.create "order create" 这条消息来分析吧！直接上源码分析：
```
/**
 * 发布一条消息
 *
 * 时间复杂度 O(N+M)，其中 N 是频道 channel 的订阅者数量，而 M 则是使用
 * 模式订阅(subscribed patterns)的客户端的数量。
 * 
 * @param channel 频道
 * @param message 消息体
 * @return 接收到信息 message 的订阅者数量
 */
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    listNode *ln;
    listIter li;

    /* 发送给监听该频道的客户端 */
    // 根据键值 channel 从字典中获取 dictEntry 对象
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        // 从 dictEntry 中获取监听 channel 的 client list
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        // 循环整个订阅消息的列表，然后发送消息
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;
            // 往指定的客户端输出缓冲区中发送消息
            // todo: 如果 client 消费消息不及时，那么 client 输出缓冲区
                    // 就会造成消息堆积，会使 redis 内存突然增大
            addReply(c,shared.mbulkhdr[3]);
            addReply(c,shared.messagebulk);
            addReplyBulk(c,channel);
            addReplyBulk(c,message);
            receivers++;
        }
    }
    /* 往监听了 channel 模式的 client 发送消息*/
    if (listLength(server.pubsub_patterns)) {
        listRewind(server.pubsub_patterns,&li);
        channel = getDecodedObject(channel);
        // 循环整个模式链表
        while ((ln = listNext(&li)) != NULL) {
            pubsubPattern *pat = ln->value;
            // 匹配指定的模式，找出指定模式对应的客户端，然后往
                    // 订阅该模式的客户端发送消息
            if (stringmatchlen((char*)pat->pattern->ptr,
                                sdslen(pat->pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) {
                // 往指定的客户端输出缓冲区中发送消息
                // todo: 如果 client 消费消息不及时，那么 client 输出缓冲区
                // 就会造成消息堆积，会使 redis 内存突然增大
                addReply(pat->client,shared.mbulkhdr[4]);
                addReply(pat->client,shared.pmessagebulk);
                addReplyBulk(pat->client,pat->pattern);
                addReplyBulk(pat->client,channel);
                addReplyBulk(pat->client,message);
                receivers++;
            }
        }
        decrRefCount(channel);
    }
    return receivers;
}
```
流程图如下：

![image.png](https://upload-images.jianshu.io/upload_images/10204326-13d1cec4db2dd160.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 四、Redis 订阅发布功能整的适合做消息中间件吗？
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过上面的分析，我想大家心里应该都已经有答案了。我们根据上面的源码分析，可以举一个小 demo，Redis 发送消息，是循环订阅者列表实现的，比如我有 100 个频道，每个频道有100个订阅者，由于是单线程，岂不是要循环处理，那么最后一个频道的最后一个订阅者岂不是会等死去。使用 redis 做消息中间件的，redis 并没有提供消息重试机制，也没有提供消息确认机制，更没有提供消息的持久化，所以一旦消息丢失，我们是没有任何办法的。而且现在突然订阅方断线，那么他将会丢失所有在短线期间发布者发布的消息，这个决定会让很多人都感到失望吧。所以还是建议大家不要使用 Redis 做消息中间件了，存在很大的风险。如果要用，还是使用强大的 RocketMQ 或 Kafka 吧。
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;文章到这里就结束了，本人水平有限，写的不好还请大家多多见谅，如有不对的地方，希望大家多提意见，我也会尽快改正。