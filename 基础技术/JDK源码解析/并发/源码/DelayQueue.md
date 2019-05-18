DelayQueue是一个无界阻塞队列，队列中每个元素都有过期时间，当队列获取元素时，只有过期元素才会出队列，队列头元素是快要过期的元素。

DelayQueue内部使用PriorityQueue存放数据，使用ReentrantLock实现线程同步，队列里面的每一个元素要实现Delayed接口，由于每个元素都有一个过期时间，所以要实现获知当前元素还剩下多少时间就过期了的接口，由于内部使用优先队列来实现，所以还要实现元素之间相互比较的接口。

![DelayQueue](../img/delayQueue.png)

```java
  /**
     * Condition signalled when a newer element becomes available
     * at the head of the queue or a new thread may need to
     * become leader.
   */
    private final Condition available = lock.newCondition();
```
条件变量available与lock锁对应，其目的是为了实现线程间的同步。

其中leader变量的使用基于Leader-Follower模式的变量，用于尽量减少不必要的线程等待。当一个线程调用队列的take方法变为leader线程后，他会调用条件变量，available.awaitNanos(delay)等待delay时间，但是其他线程（flower线程）则会调用available.await()进行无限等待，leader
线程延迟时间过期后，会退出take方法，并通过调用available.string()方法唤醒一个follwer线程，被唤醒的follwer线程选举为新的leader线程。

#### offer（）
插入元素到队列，如果插入的元素为null则空指针，否则由于是无界队列，所以一直返回True，插入元素要实现Delayed接口
```java


    /**
     * Inserts the specified element into this delay queue.
     *
     * @param e the element to add
     * @return {@code true}
     * @throws NullPointerException if the specified element is null
     */
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

首先获取独占锁，然后田间 元素到优先级队列，由于q是优先级队列，所以添加元素后，调用q.peek返回的并不一定是当前添加的元素，如果返回为true，则说明当前元素是最先将过期的，那么重置leader线程为null，这个时候激活avalable变量条件队列里面的一个线程，告诉他队列里面有元素了。

```
#### take操作
获取并移除队列里面延迟时间过期的元素，如果队列没有过期元素则等待。
```java

   /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue.
     *
     * @return the head of this queue
     * @throws InterruptedException {@inheritDoc}
     */
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```
首先会先获取独占锁lock，假设线程A第一次调用队列的take（）方法时队列为空，则执行代码first=null，所以会把当前线程放入available的条件队列里面阻塞等待。

当有另一个线程B执行offer（item）并且添加元素到队列的时候，假设此时没有其他线程执行入队操作，则线程B添加的元素是对手元素，那么执行q.peek（）；

e这个时候就会重置leader线程为null，并且激活条件变量队列里面的一个线程，此时线程A就会被激活。

线程A被激活并循环后重新获取队首元素，这时候first就是线程B新增的元素，可知这个时候first不为null，则调用first.getDelay(TimeUtil.NANOSECONDS)方法查看该元素还剩多少时间就要过期，如果delay<=0则说明已经过期，直接出队返回，否则查看
leader是否为null，不为null则说明其他线程也在执行take，则把该线程放入条件队列，如果这个时候leader不为null，则选取当前A为Leader线程，然后等待Delay时间，（于此期间，该线程会释放锁，所以其他线程可以offer添加元素，也可以阻塞自己）剩余时间到期后，线程A会竞争重新得到锁，然后重置leader线程为nulll，重新进入循环，此时会发现队头元素已经过期了，所以会直接返回队头元素。


在返回前会执行finally块里面的代码，如果执行结果为true，则说明当前线程从队列里面移除过期元素后，又有其他线程执行了入队操作，那么此时调用条件变量的singal方法，激活条件队列里面的等待线程。

#### poll
获取并移除队头过期元素。

```java

    /**
     * Retrieves and removes the head of this queue, waiting if necessary
     * until an element with an expired delay is available on this queue,
     * or the specified wait time expires.
     *
     * @return the head of this queue, or {@code null} if the
     *         specified waiting time elapses before an element with
     *         an expired delay becomes available
     * @throws InterruptedException {@inheritDoc}
     */
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null) {
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    if (nanos <= 0)
                        return null;
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }

```
首先获取独占锁，然后获取队头元素，如果队头元素为null或者还没过期则返回null，否则返回队头元素。

