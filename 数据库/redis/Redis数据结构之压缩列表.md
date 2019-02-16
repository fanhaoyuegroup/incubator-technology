### 一、什么是压缩列表

压缩列表（ziplist）是 Redis 为了节约内存而开发的，由一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。一个压缩列表可以包含任意多个节点（`entry`）， 每个节点可以保存一个字节数组或者一个整数值。

下面是一个关于压缩列表的构成图：
![](http://phtpnyqb4.bkt.clouddn.com/ziplist1.png)

 - `zlbytes`：记录整个压缩列表占用的内存字节数
 - `ztail`：记录压缩列表表尾节点距离起始地址有多少字节
 - `zllen`：记录了压缩列表包含的节点数量
 - `entryX`：压缩链表各个节点
 - `zlend`：特殊值（0xFF），用于标记压缩列表的末端

例子解读：
![](http://phtpnyqb4.bkt.clouddn.com/ziplist.png)
 
### 二、压缩列表节点

![](http://phtpnyqb4.bkt.clouddn.com/ziplist3.png)
压缩列表节点构成如上图，下面对每个构成分别介绍。

**2.1 `previous_entry_length`**

以字节为单位，记录了压缩链接前一个节点的长度。`previous_entry_length` 属性的长度可以是 1 字节或者 5 字节。

- 如果前一节点的长度小于 254 字节， 那么 `previous_entry_length` 属性的长度为 1 字节： 前一节点的长度就保存在这一个字节里面
- 如果前一节点的长度大于等于 254 字节， 那么 `previous_entry_length` 属性的长度为 5 字节

因为这个属性记录了前一个节点的长度，所以程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址。

压缩列表从尾到头的遍历原理：指向某个节点地址的指针，那么通过这个指针以及这个节点的 `previous_entry_length` 属性，程序就可以一直向前一个节点回溯， 最终到达压缩列表的表头节点。

**2.2 `encoding`**

节点的 `encoding` 属性记录了节点的 `content` 属性所保存数据的类型以及长度。

**2.3 `content`**

节点的 `content` 属性负责保存节点的值，节点值可以是一个字节数组或者整数， 值的类型和长度由节点的 `encoding` 属性决定。

例子解读：
![](http://phtpnyqb4.bkt.clouddn.com/ziplist4.png)

### 三、连锁更新

考虑这样一种情况： 在一个压缩列表中， 有多个连续的、长度介于 250 字节到 253 字节之间的节点 e1 至 eN。

因为 e1 至 eN 的所有节点的长度都小于 254 字节，所以记录这些节点的长度只需要 1 字节长的 `previous_entry_length` 属性， 换句话说， e1 至 eN 的所有节点的 `previous_entry_length` 属性都是 1 字节长的。

如果我们将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点， 那么 new 将成为 e1 的前置节点。

![](http://phtpnyqb4.bkt.clouddn.com/ziplist5.png)

因为 e1 的 `previous_entry_length` 属性仅长 1 字节， 它没办法保存新节点 new 的长度， 所以程序将对压缩列表执行空间重分配操作， 并将 e1 节点的 `previous_entry_length`h` 属性从原来的 1 字节长扩展为 5 字节长。

现在， 麻烦的事情来了，e1 原本的长度介于 250 字节至 253 字节之间， 在为 `previous_entry_length` 属性新增四个字节的空间之后， e1 的长度就变成了介于 254 字节至 257 字节之间， 而这种长度使用 1 字节长的 `previous_entry_length` 属性是没办法保存的。

因此， 为了让 e2 的 `previous_entry_length` 属性可以记录下 e1 的长度， 程序需要再次对压缩列表执行空间重分配操作， 并将 e2 节点的 `previous_entry_length` 属性从原来的 1 字节长扩展为 5 字节长。

正如扩展 e1 引发了对 e2 的扩展一样， 扩展 e2 也会引发对 e3 的扩展， 而扩展 e3 又会引发对 e4 的扩展……为了让每个节点的 `previous_entry_length` 属性都符合压缩列表对节点的要求，程序需要不断地对压缩列表执行空间重分配操作， 直到 eN 为止。