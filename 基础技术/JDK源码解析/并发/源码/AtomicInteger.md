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
