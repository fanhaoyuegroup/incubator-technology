前面AtomicBoolean中对原子更新值已经讲得差不多了，AtomicInteger实现的核心也跟AtomicBoolean几乎是一样的，不过AtomicInteger相比起来多了几个方法，我还是理一遍，AtomicBoolean中有的就不过多解释了。
#### 1.AtomicInteger是什么
- 一个可以原子方式更新的{@code int}值。
- {@code AtomicInteger}用于诸如原子递增计数器的应用程序中，但是不能用作{@link java.lang.Integer}的替代。
- 继承了java.lang.Number以允许通过处理基于数字的类的工具和实用程序进行统一访问。
#### 2. AtomicBoolean 内部的属性
~~~ java
// 设置为使用Unsafe.compareAndSwapInt进行更新
private static final Unsafe unsafe = Unsafe.getUnsafe();
//保存修改变量的实际内存地址，通过unsafe.objectFieldOffset读取
private static final long valueOffset;

 // 初始化的时候,执行静态代码块，计算出保存的value的内存地址便于直接进行内存操作
 //objectFieldOffset(Final f):返回给定的非静态属性在它的类的存储分配中的位置(偏移地址)。
static {
    try { 
            valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));  
        } catch (Exception ex) {
            throw new Error(ex);
        }
}
private volatile int value;
~~~
#### 3.构造函数
~~~ java
/** 
* Creates a new AtomicInteger with the given initial value.
* 
* @param initialValue the initial value
*/
public AtomicInteger(int initialValue)
 { 
    value = initialValue;
}


/**
* Creates a new AtomicInteger with initial value {@code 0}.
*/
public AtomicInteger() {}
~~~
#### 4.get方法
~~~ java
/** 
* Gets the current value. 
* 
* @return the current value 
*/
public final int get() {    
    return value;
}

/** 
* Atomically sets to the given value and returns the old value. 
*
* @param newValue the new value 
* @return the previous value 
*/
public final int getAndSet(int newValue) { 
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}

public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, 
update);
}

public final boolean weakCompareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, 
update);
}
~~~
##### AtomicInteger有，而AtomicBoolean没有的get方法
~~~ java
/** 
* Atomically increments by one the current value. 
* 以原子方式将当前值增加1。返回当前值
* @return the previous value 
*/
public final int getAndIncrement() { 
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

/** 
* Atomically decrements by one the current value. 
* 原子地将当前值减1。返回当前值
* @return the previous value 
*/
public final int getAndDecrement() {
    return unsafe.getAndAddInt(this, valueOffset, -1);
}

/** 
* Atomically adds the given value to the current value. 
* 以原子方式将给定值添加到当前值。返回当前值
* @param delta the value to add 
* @return the previous value 
*/
public final int getAndAdd(int delta) { 
    return unsafe.getAndAddInt(this, valueOffset, delta);
}

/** 
* Atomically increments by one the current value. 
* 以原子的方式将当前值加1，并返回增加后的值
* @return the updated value 
*/
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

/** 
* Atomically decrements by one the current value. 
* 原子地将当前值减1。返回减1后的值
* @return the updated value 
*/
public final int decrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
}

/**
* Atomically adds the given value to the current value.
* 以原子方式将给定值添加到当前值。返回增加后的值
* @param delta the value to add
* @return the updated value
*/
 public final int addAndGet(int delta) {
     return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
 }

//用给定函数的结果作为更新值，并返回更新前的值
 public final int getAndUpdate(IntUnaryOperator updateFunction) {
     int prev, next;
     do {
         prev = get();
         //运算后的结果给到next
         next = updateFunction.applyAsInt(prev);
     } while (!compareAndSet(prev, next));
     return prev;
 }

//用给定函数的结果作为更新值，并返回更新后的值
 public final int updateAndGet(IntUnaryOperator updateFunction) {
    int prev, next;
    do {
        prev = get(); 
        next = updateFunction.applyAsInt(prev);
     } while (!compareAndSet(prev, next));
     return next;
 }
 //将指定函数应用于当前值和给定值，并将结果更新为当前值，返回更新前的值
public final int getAndAccumulate(int x,IntBinaryOperator accumulatorFunction {    
    int prev, next;
    do {
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));    
    return prev;
}

//将指定函数应用于当前值和给定值，并将结果更新为当前值，返回更新后的值
public final int accumulateAndGet(int x,IntBinaryOperator accumulatorFunction) {   
    int prev, next;
    do {  
        prev = get();
        next = accumulatorFunction.applyAsInt(prev, x);
    } while (!compareAndSet(prev, next));
    return next;
}
~~~
源码中的CAS操作，全是调用的Unsafe中的CAS方法，Unsafe中的CAS操作都是native方法
#### 5.AtomicInteger的使用举栗
~~~ java
//线程安全自增
private AtomicInteger atomicInteger = new AtomicInteger();
public void incr() {
    atomicInteger.incrementAndGet();
}
~~~
