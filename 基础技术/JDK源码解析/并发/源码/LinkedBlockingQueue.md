## `LinkedBlockingQueue`

### 1.1 类继承关系

``` java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {}
```

- 说明：LinkedBlockingQueue继承了AbstractQueue抽象类，AbstractQueue定义了对队列的基本操作；同时实现了BlockingQueue接口，BlockingQueue表示阻塞型的队列，其对队列的操作可能会抛出异常；同时也实现了Searializable接口，表示可以被序列化。

### 1.2 类的内部类

``` java
static class Node<E> {
    // 元素
        E item;
    // next域
    Node<E> next;
    // 构造函数
        Node(E x) { item = x; }
    }
```

- 说明：Node类非常简单，包含了两个域，分别用于存放元素和指示下一个结点。

### 1.3 类属性

``` java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    // 版本序列号
    private static final long serialVersionUID = -6903933977591709194L;
    // 容量
    private final int capacity;
    // 元素的个数
    private final AtomicInteger count = new AtomicInteger();
    // 头结点
    transient Node<E> head;
    // 尾结点
    private transient Node<E> last;
    // 取元素锁
    private final ReentrantLock takeLock = new ReentrantLock();
    // 非空条件
    private final Condition notEmpty = takeLock.newCondition();
    // 存元素锁
    private final ReentrantLock putLock = new ReentrantLock();
    // 非满条件
    private final Condition notFull = putLock.newCondition();
}
```

- 说明：可以看到LinkedBlockingQueue包含了读、写重入锁（与ArrayBlockingQueue不同，ArrayBlockingQueue只包含了一把重入锁），读写操作进行了分离，并且不同的锁有不同的Condition条件（与ArrayBlockingQueue不同，ArrayBlockingQueue是一把重入锁的两个条件）。

### 1.4 构造函数

- 无参构造函数：容量默认为Integer的最大值
``` java
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }
```

- 单参构造函数
``` java
public LinkedBlockingQueue(int capacity) {
        // 初始化容量必须大于0
        if (capacity <= 0) throw new IllegalArgumentException();
        // 初始化容量
        this.capacity = capacity;
        // 初始化头结点和尾结点
        last = head = new Node<E>(null);
    }
```

- 集合参构造函数：此构造函数注意点：`集合里不能有null元素，否则会抛异常`
``` java
public LinkedBlockingQueue(Collection<? extends E> c) {
        // 调用重载构造函数
        this(Integer.MAX_VALUE);
        // 存锁
        final ReentrantLock putLock = this.putLock;
        // 获取锁
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) { // 遍历c集合
                if (e == null) // 元素为null,抛出异常
                    throw new NullPointerException();
                if (n == capacity) // 
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

### 1.5 核心函数分析

- put函数

``` java
public void put(E e) throws InterruptedException {
        // 值不为空
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        // 
        int c = -1;
        // 新生结点
        Node<E> node = new Node<E>(e);
        // 存元素锁
        final ReentrantLock putLock = this.putLock;
        // 元素个数
        final AtomicInteger count = this.count;
        // 如果当前线程未被中断，则获取锁
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) { // 元素个数到达指定容量
                // 在notFull条件上进行等待
                notFull.await();
            }
            // 入队列
            enqueue(node);
            // 更新元素个数，返回的是以前的元素个数
            c = count.getAndIncrement();
            if (c + 1 < capacity) // 元素个数是否小于容量
                // 唤醒在notFull条件上等待的某个线程
                notFull.signal();
        } finally {
            // 释放锁
            putLock.unlock();
        }
        if (c == 0) // 元素个数为0，表示已有take线程在notEmpty条件上进入了等待，则需要唤醒在notEmpty条件上等待的线程
            signalNotEmpty();
    }
```
- 说明：put函数用于存放元素，其流程如下。
　　① 判断元素是否为null，若是，则抛出异常，否则，进入步骤②
　　② 获取存元素锁，并上锁，如果当前线程被中断，则抛出异常，否则，进入步骤③
　　③ 判断当前队列中的元素个数是否已经达到指定容量，若是，则在notFull条件上进行等待，否则，进入步骤④
　　④ 将新生结点入队列，更新队列元素个数，若元素个数小于指定容量，则唤醒在notFull条件上等待的线程，表示可以继续存放元素。进入步骤⑤
　　⑤ 释放锁，判断结点入队列之前的元素个数是否为0，若是，则唤醒在notEmpty条件上等待的线程（表示队列中没有元素，取元素线程被阻塞了）。
  
- offer函数

```java
public boolean offer(E e) {
        // 确保元素不为null
        if (e == null) throw new NullPointerException();
        // 获取计数器
        final AtomicInteger count = this.count;
        if (count.get() == capacity) // 元素个数到达指定容量
            // 返回
            return false;
        // 
        int c = -1;
        // 新生结点
        Node<E> node = new Node<E>(e);
        // 存元素锁
        final ReentrantLock putLock = this.putLock;
        // 获取锁
        putLock.lock();
        try {
            if (count.get() < capacity) { // 元素个数小于指定容量
                // 入队列
                enqueue(node);
                // 更新元素个数，返回的是以前的元素个数
                c = count.getAndIncrement();
                if (c + 1 < capacity) // 元素个数是否小于容量
                    // 唤醒在notFull条件上等待的某个线程
                    notFull.signal();
            }
        } finally {
            // 释放锁
            putLock.unlock();
        }
        if (c == 0) // 元素个数为0，则唤醒在notEmpty条件上等待的某个线程
            signalNotEmpty();
        return c >= 0;
    }
```
- 说明：offer函数也用于存放元素，offer函数添加元素不会抛出异常（其他的和put函数类似）。

- take函数

```java
public E take() throws InterruptedException {
        E x;
        int c = -1;
        // 获取计数器
        final AtomicInteger count = this.count;
        // 获取取元素锁
        final ReentrantLock takeLock = this.takeLock;
        // 如果当前线程未被中断，则获取锁
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) { // 元素个数为0
                // 在notEmpty条件上等待
                notEmpty.await();
            }
            // 出队列
            x = dequeue();
            // 更新元素个数，返回的是以前的元素个数
            c = count.getAndDecrement();
            if (c > 1) // 元素个数大于1，则唤醒在notEmpty上等待的某个线程
                notEmpty.signal();
        } finally {
            // 释放锁
            takeLock.unlock();
        }
        if (c == capacity) // 元素个数到达指定容量
            // 唤醒在notFull条件上等待的某个线程
            signalNotFull();
        // 返回
        return x;
    }
```
- 说明：take函数用于获取一个元素，其与put函数相对应，其流程如下。
　　① 获取取元素锁，并上锁，如果当前线程被中断，则抛出异常，否则，进入步骤②
　　② 判断当前队列中的元素个数是否为0，若是，则在notEmpty条件上进行等待，否则，进入步骤③
　　③ 出队列，更新队列元素个数，若元素个数大于1，则唤醒在notEmpty条件上等待的线程，表示可以继续取元素。进入步骤④
　　④ 释放锁，判断结点出队列之前的元素个数是否为指定容量，若是，则唤醒在notFull条件上等待的线程（表示队列已满，存元素线程被阻塞了）。

- poll函数

```java
public E poll() {
        // 获取计数器
        final AtomicInteger count = this.count;
        if (count.get() == 0) // 元素个数为0
            return null;
        // 
        E x = null;
        int c = -1;
        // 取元素锁
        final ReentrantLock takeLock = this.takeLock;
        // 获取锁
        takeLock.lock();
        try {
            if (count.get() > 0) { // 元素个数大于0
                // 出队列
                x = dequeue();
                // 更新元素个数，返回的是以前的元素个数
                c = count.getAndDecrement();
                if (c > 1) // 元素个数大于1
                    // 唤醒在notEmpty条件上等待的某个线程
                    notEmpty.signal();
            }
        } finally {
            // 释放锁
            takeLock.unlock();
        }
        if (c == capacity) // 元素大小达到指定容量
            // 唤醒在notFull条件上等待的某个线程
            signalNotFull();
        // 返回元素
        return x;
    }
```
- 说明：poll函数也用于获取元素，poll函数添加元素不会抛出异常（其他的与take函数类似）