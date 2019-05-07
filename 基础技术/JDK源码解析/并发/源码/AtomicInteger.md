#### 1.AtomicInteger 是什么

- 一个可以原子方式更新的 int 值。
- `AtomicInteger` 用于诸如原子递增计数器的应用程序中，但是不能用作 `java.lang.Integer` 的替代。
- 继承了 `java.lang.Number` 以允许通过处理基于数字的类的工具和实用程序进行统一访问。

#### 2. AtomicBoolean 内部的属性

``` java
// 设置为使用 Unsafe.compareAndSwapInt 进行更新
private static final Unsafe unsafe = Unsafe.getUnsafe();
//保存修改变量的实际内存地址，通过 unsafe.objectFieldOffset 读取
private static final long valueOffset;

 // 初始化的时候,执行静态代码块，计算出保存的 value 的内存地址便于直接进行内存操作
 //objectFieldOffset(Final f):返回给定的非静态属性在它的类的存储分配中的位置 (偏移地址)。
static {
    try { 
            valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));  
        } catch (Exception ex) {
            throw new Error(ex);
        }
}
private volatile int value;
```
#### 3.构造函数

``` java
public AtomicInteger(int initialValue) { 
    value = initialValue;
}

public AtomicInteger() {}
```

#### 4.getAndIncrement 方法

``` java
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
```
源码中的 CAS 操作，全是调用的 `Unsafe` 中的 CAS 方法，`Unsafe` 中的 CAS 操作都是 native 方法。下面来看一下 `Unsafe` 类中的 `getAndAddInt` 方法。

``` java
    /**
     * @param var1 当前 AtomicInteger 对象的引用
     * @param var2 value 在内存中的偏移量
     * @param var4 增加的值
     */
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        // 当预期值与内存值相等时结束循环
        do {
            // 获取内存中的 value，当作预期值进行循环判断
            var5 = this.getIntVolatile(var1, var2);
            /**
             * @param var5        预期值
             * @param var5 + var4 更新值
             */
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

#### 5.AtomicInteger 的使用举栗

``` java
//线程安全自增
private AtomicInteger atomicInteger = new AtomicInteger();
public void incr() {
    atomicInteger.incrementAndGet();
}
```

#### 1.CAS做了什么？
CAS有三个操作数：内存值V，旧的预期值A，需要修改的新值B
CAS涉及两个步骤：
  - 1.compare：比较内存值是否与预期值A相等。
  - 2.set：如果内存值与预期值A相等，set值为B。
虽然这是两步，但是CAS保证了这两步操作的原子性，因此可以将上面两步视为一个原子操作。

#### 2.处理器如何实现原子操作？
**1.使用总线锁保证原子性**
总线锁就是使用处理器提供的一个LOCK＃信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住,那么该处理器可以独占使用共享内存。

**缺点：** 效率低
**原因：** 总线锁把CPU和内存之间的通信锁住了，导致锁定期间其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大。

**2.使用缓存锁保证原子性**
缓存锁就是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，那么当它执行锁操作会写内存时，处理器不在总线上声言LOCK#信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会组织同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效。

#### 3.CAS缺点
- **循环时间长开销很大**
    如果CAS失败，会一直进行尝试。如果CAS长时间一直不成功，可能会给CPU带来很大的开销。
- **只能保证一个共享变量的原子操作**
- **ABA问题**

#### 4.ABA问题
如果从内存地址V取值，原来值为A，修改为B，之后又将值修改为A，CAS会认为该值从来没变过。
解决思路：加版本号。