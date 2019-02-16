### 一、Redis数据结构之动态字符串

在这里主要将redis的字符串与c做比较，来认识一下sds

```text
         C字符串 	                                  SDS
- 获取字符串长度的复杂度为O（n）   	            获取字符串长度的复杂度O（1）
- API是不安全的，可能造成缓冲区溢出	                   API是安全的
- 修改字符串长度N次，必然要执行N次内存重分配 	  修改字符长度N次，最多执行N次内存重分配
- 只能保存文本数据	                                可以保存文本或者二进制数据
- 可使用<string.h>库中的函数  	                可使用一部分<string.h>库中的函数

```

1.sds动态字符串的结构

c语言字符串结构单一，是一个字符数组，获取长度需从头到尾遍历，故获取字符串长度的复杂度为O（n），而redis的sds结构中已定义字符串长度，故获取字符串长度的复杂度O（1）。
```
/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```


2.避免缓冲区溢出的操作

- c不记录字符串的长度，很容易造成缓冲区溢出;
- SDS的空间分配策略安全杜绝了发生缓冲区溢出的可能性，当对SDS进行修改时，会先检查SDS的空间是否满足需要，如果不满足则会自动扩展至执行修改所需的大小(属性len增加)，然后才执行实际的修改操作,在后面的空间分配中会说明。

```
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);
    //追加时先进行扩容，后面详细说明
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    //拼接字符串
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}
// s:原数组     
//strlen(t) 需拼接的目标数组的长度
sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}
```
3.内存重分配问题

空间预留分配：对SDS进行修改时对SDS进行空间扩展的同时，还会对SDS分配额外的未使用的空间

``` 
//redis的扩容
 sds sdsMakeRoomFor(sds s, size_t addlen) {
     void *sh, *newsh;
     size_t avail = sdsavail(s);
     size_t len, newlen;
     char type, oldtype = s[-1] & SDS_TYPE_MASK;
     int hdrlen;
 
     /* Return ASAP if there is enough space left. */
     if (avail >= addlen) return s;
 
     len = sdslen(s);
     sh = (char*)s-sdsHdrSize(oldtype);
     //新的字符串长度 = 旧len+新addlen，至少分配这么多
     newlen = (len+addlen);
     //预分配策略
     if (newlen < SDS_MAX_PREALLOC)
     //1.若小于SDS_MAX_PREALLOC(1M)，则分配现在长度的2倍
         newlen *= 2;
     else
     //2.若大于SDS_MAX_PREALLOC(1M)，则分配现在长度+SDS_MAX_PREALLOC(1M)
         newlen += SDS_MAX_PREALLOC;
 
    ......
     return s;
 }

```

 - char类型不能高效的支持长度计算和追加两种操作：对字符串进行N次追加，必定需要对字符串进行N次内存重分配。           
 - sds会为追加操作进行优化：加快追加操作的速度，并降低内存分配的次数，代价是多占用了一些内存，而且这些内存不会被主动释放( buf分配额外空间，可以让追加操作内存重分配的次数大大减少)。
 
 4.惰性空间释放
 
 惰性空间释放用于优化 SDS 的字符串缩短操作： 当 SDS 的 API 需要缩短 SDS 保存的字符串时， 程序并不立即使用内存重分配来回收缩短后多出来的字节， 而是使用 free 属性将这些字节的数量记录起来， 并等待将来使用。



