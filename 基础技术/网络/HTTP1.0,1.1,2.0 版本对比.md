### HTTP1.0

HTTP1.0 规定浏览器与服务器保持短暂的连接，浏览器每次请求服务器都会创建一个 TCP 连接，服务器响应后就会断开 TCP 连接。由于每次请求都会建立 TCP 连接，而 TCP 连接的建立与释放需要经历三次握手与四次挥手阶段，因此网络的利用率非常低。

HTTP1.0 规定下一个请求必须在前一个请求到达之前才能发送。如果前一个请求的响应一直没有到达，后面的请求就会阻塞。

### HTTP1.1

HTTP1.1 使用长连接与流水线方式解决了 HTTP1.0 每次请求都重新建立 TCP 连接的问题，使一个 TCP 连接可以传送多个 HTTP 的请求与响应，减小建立与断开连接的消耗。（HTTP1.1 请求头 Connection 字段）

缓存处理，在 HTTP1.0 中主要使用 header 里的 If-Modified-Since，Expires 来做为缓存判断的标准，HTTP1.1 则引入了更多的缓存控制策略例如 Entity tag，If-Unmodified-Since，If-Match，If-None-Match 等更多可供选择的缓存头来控制缓存策略。

增加 host 请求头域，用于实现虚拟主机技术。

HTTP1.1 中新增了 24 个错误状态响应码。

新增 range 头域实现断点续传。

### HTTP2.0

HTTP2.0 通过在应用层和传输层之间增加一个二进制分帧层，突破了 HTTP1.1 的性能限制、改进传输性能。

![](https://image-static.segmentfault.com/293/521/293521278-5a6e83af997f9_articlex)

多路复用，客户端向某个域名的服务器请求页面的过程中，只会创建一个 TCP 连接。每个请求是一个数据流，数据流以消息的方式发送，而消息又分为多个帧，帧头部记录着 stream id 用来标识所属的数据流，不同属的帧可以在连接中随机混杂在一起。接收方可以根据 stream id 将帧再归属到各自不同的请求当中去。

头部压缩，HTTP2.0 使用 encoder 来减少需要传输的 header 大小，通讯双方各自 cache 一份 header fields 表，既避免了重复 header 的传输，又减小了需要传输的大小。高效的压缩算法可以很大程度地压缩 header，减少发送包的数量从而降低延迟。

服务器推送，服务器还可以额外的向客户端推送资源，而无需客户端明确的请求。

### 参考

[HTTP1.0 HTTP1.1 HTTP2.0 主要特性对比](https://segmentfault.com/a/1190000013028798) by Steven <br>
[HTTP1.0、HTTP1.1 和 HTTP2.0 的区别](https://mp.weixin.qq.com/s/GICbiyJpINrHZ41u_4zT-A) by  code小生 <br>