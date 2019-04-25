## `PriorityBlockingQueue`

一、`PriorityBlockingQueue`的主要属性
```java
/**
 * 入队、出队、扩容等操作的锁
 */
private final ReentrantLock lock;

/**
 * 为空的时候用来充当阻塞条件
 */
private final Condition notEmpty;

/**
 * 自旋锁 0 1控制
 */
private transient volatile int allocationSpinLock;

/**
 * 兼容版本使用队列，主要用于序列化的时候
 */
private PriorityQueue<E> q;
```

二、`构造函数`
1. 无参构造函数：默认初始化大小11，无比较器
2. 单参构造函数：用户自定义初始化大小，无比较器
3. 双参构造函数：用户自定义初始化大小，有比较器
4. 传递Collection完成入队构造函数

三、`主要使用的函数分析`
1. add方法，主要调用offer函数，所以我们分析offer函数

```java
public boolean offer(E e) {
    // 不允许往队列添加Null元素
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    // 是否需要扩容  如果元素个数要比容量还大的时候就要扩容了
    while ((n = size) >= (cap = (array = queue).length))
        // 扩容
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        // 如果无比较器 使用默认实现comparable接口的方法比较
        // 反之就采用比较器比较，这两个方法都是在做调整二叉堆结构
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();
    } finally {
        lock.unlock();
    }
    return true;
}
```
- offer函数分析：
    * 1、判断元素null
    * 2、加锁
    * 3、判断是否 `扩容`（循环操作）
    * 4、判断是否有比较器
    * 5、调整结构
    * 6、唤醒堵塞的消费线程
    * 7、释放锁
    * 8、offer函数还是很容易理解的，最难理解的扩容函数
    
2. ` tryGrow `方法

```java
private void tryGrow(Object[] array, int oldCap) {
    // 这里要释放原来获取的锁，先扩容才可以再添加元素，
    // todo：这里如果释放掉锁的话，很可能会有很多线程都来扩容，那现在应该怎们办呢？
    // 解答：使用 CAS 操作，只让一个线程进行扩容操作；
    lock.unlock(); // must release and then re-acquire main lock
    Object[] newArray = null;
    // todo：这里采用 cas 的操作来扩容, 使用 CAS 是为了保证同一时刻只有一个线程在做扩容操作
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                 0, 1)) {
        try {
            // todo：这里为啥要以 64 为标准？
            int newCap = oldCap + ((oldCap < 64) ?
                                   (oldCap + 2) : 
                                   (oldCap >> 1));
            if (newCap - MAX_ARRAY_SIZE > 0) {    
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            // 重新将值给位 1，以便后面线程来扩容
            allocationSpinLock = 0;
        }
    }
    // 如果 newArray == null 说明有其他线程正在这里扩容，因为我们发现上面的逻辑，如果有线程进行扩容的话，会将 allocationSpinLockOffset 置为 1
    if (newArray == null)
        Thread.yield();
    // 然后将原来的元素重新赋值到新的数组里面，那么该操作是需要重新加锁的
    // queue == array???
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```
- 扩容函数分析
    * 1、释放之前当前线程调用offer函数拿到锁
    * 2、cas操作执行扩容
    * 3、扩容规则：旧的容量小与64 则是old+old+2 否则是old+old乘以2，然后判断是否大于max_size，然后执行new Object[newCap]操作
    * 4、将cas标志位归0
    * 5、判断newArray新数组是否执行了new操作，如果这个线程没有执行扩容，那么就执行yield函数让步(并不一定就会让出执行权)
    * 6、加锁
    * 7、执行扩容
    * 8、复制数组元素
    
3. take函数（可中断、并且会堵塞）、poll函数不会抛异常、不会堵塞。内部都是调用dequeue函数
4. 出队有多个方法：poll、poll带超时时间、take、remove

```java
 public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            //如果出队的元素为null 那么就堵塞了，等待元素入队
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 仅仅执行出队操作
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

```java
private E dequeue() {
    int n = size - 1;
    if (n < 0) {
        return null;
    }
    Object[] array = queue;
    E result = (E) array[0];
    E x = (E) array[n];
    array[n] = null;
    Comparator<? super E> cmp = comparator;
    // 重新调整堆结构
    if (cmp == null)
        // 没有比较器的时候
        siftDownComparable(0, x, array, n);
    else
        // 有比较器的时候
        siftDownUsingComparator(0, x, array, n, cmp);
    size = n;
    return result;
}
```
四、`总结`

1. 适合在生产消费模型业务中使用
2. 出队操作一定要选择好，因为有出队堵塞、出队不堵塞的、抛异常
3. 线上问题从` 扩容函数 `分析
4. [并发队列使用排查分析](https://juejin.im/post/5cad3600f265da03761e6d63)
    