#### 1.AtomicBoolean是什么
- 一个可以原子方式更新的{@code boolean}值。
- 该值可以作为原子更新的标志，但是不能用作java.lang.Boolean的替代。
#### 2. AtomicBoolean 内部的属性
``` java
// 设置为使用Unsafe.compareAndSwapInt进行更新
private static final Unsafe unsafe = Unsafe.getUnsafe();
//保存修改变量的实际内存地址，通过unsafe.objectFieldOffset读取
private static final long valueOffset;

 // 初始化的时候,执行静态代码块，计算出保存的value的内存地址便于直接进行内存操作
 //objectFieldOffset(Final f):返回给定的非静态属性在它的类的存储分配中的位置(偏移地址)。
static {
    try { 
            valueOffset = unsafe.objectFieldOffset  
                (AtomicBoolean.class.getDeclaredField("value"));  
        } catch (Exception ex) {
            throw new Error(ex);
        }
}
private volatile int value;
```
#### 3.构造函数
``` java
/** 
* Creates a new {@code AtomicBoolean} with the given initial value.
* 通过给定的初始值，将boolean转为int后初始化value
* @param initialValue the initial value 
*/
public AtomicBoolean(boolean initialValue) { 
    value = initialValue ? 1 : 0;
}


/**
* Creates a new {@code AtomicBoolean} with initial value {@code false}. 
* 初始化为默认值，默认为false，因为int的默认值是0
*/
public AtomicBoolean() {}
```
#### 4.get方法
``` java
/** 
* Returns the current value. 
*
* @return the current value
*/
public final boolean get() {
    return value != 0;
}

/** 
* Atomically sets to the given value and returns the previous value. 
* 
* @param newValue the new value 
* @return the previous value 
*/
// 通过原子的方式设置给定的值，并返回之前的值
public final boolean getAndSet(boolean newValue) {
    boolean prev;
    do {
        //先get()到原来的值，再进行原子更新，会一直循环知道更新成功
        prev = get(); 
    } while (!compareAndSet(prev, newValue));
    return prev;
}
```
这边主要是调用了compareAndSet(boolean expect, boolean update)来设置值，我们就先看看这个方法
``` java
/**
* Atomically sets the value to the given updated value 
* if the current value {@code ==} the expected value. 
* 
* @param expect the expected value 
* @param update the new value 
* @return {@code true} if successful. False return indicates that 
* the actual value was not equal to the expected value. 
*/
public final boolean compareAndSet(boolean expect, boolean 
update) {
    int e = expect ? 1 : 0;
    int u = update ? 1 : 0;
    //unsafe.compareAndSwapInt:原子性地更新偏移地址为valueOffset的属性值为u，当且仅当偏移地址为alueOffset的属性的当前值为e才会更新成功，否则返回false。
    return unsafe.compareAndSwapInt(this, valueOffset, e, u);
}
```
#### 5.set方法
``` java
/**
* Unconditionally sets to the given value. 
*
* @param newValue the new value 
*/
//无条件设置为给定值
public final void set(boolean newValue) {
    value = newValue ? 1 : 0;
}

/**
* Eventually sets to the given value. 
* 
* @param newValue the new value 
* @since 1.6 
*/
//最终设置为给定值
public final void lazySet(boolean newValue) {
    int v = newValue ? 1 : 0;
    //unsafe.putOrderedInt(this, valueOffset, v):根据偏移地址，设置对应的值为指定值v。这是一个有序或者有延迟的putIntVolatile()方法，并且不保证值的改变被其他线程立即看到。只有在field被volatile修饰并且期望被修改的时候使用才会生效。
    unsafe.putOrderedInt(this, valueOffset, v);
}
```
这个类的方法不多，基本上上个面都列举到了。这个实现原子性的主要核心都是在Unsafe里面。有兴趣的话，可以把这个类的源码拉来看看。
#### 5.AtomicBoolean的使用举栗
``` java
//如果想让某种操作只执行一次,初始atomicBoolean为false
AtomicBoolean atomicBoolean = new AtomicBoolean(false);
//如果当前值为false，设置当前值为true，如果设置成功，返回true
if (atomicBoolean.compareAndSet(false,true)){
    //执行操作
}
```




参考文档：[JAVA中神奇的双刃剑--Unsafe](https://www.cnblogs.com/throwable/p/9139947.html)