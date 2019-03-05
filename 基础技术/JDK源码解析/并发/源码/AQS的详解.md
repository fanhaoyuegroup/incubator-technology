# AQS的详解
## 前言
#### java的JUC包我断断续续看了零星的几个实现类，一直想写一些文章记录一下，但是由于种种原因，一是由于自己的懒惰，也因为作为一个新手，这种东西我实在没太好的文笔能表达清楚，今天我再次想冲击一下这个疑难杂症。首先说一下，AQS是一个java的java.util.concurrent包中的一个名为AbstractQueuedSynchronizer的类，简称AQS。那这个类为什么这么重要呢？因为这个JUC包中包含了几乎所有我们平时会用于控制并发的锁和并发控制类，而这个AQS更是基本所有锁的核心控制框架，所以这个类是我们学习并发的重中之重。
## 写在前面
#### 在本系列的后面的文章中，我们会讲到，当多个线程抢占较少的资源的时候，AQS内部所做的一些关于分配资源的策略方法。在这之前我建议大家先想一下，如果由自己设计一个最简单的抢占模型，我们会怎么设计？我这边设想了一下，肯定会有下面几个步骤
	1: 提供一个资源供大家抢占，假设初始状态为1，被占用后设置为0.所以假设有多个线程去获取这个资源的时候，肯定会只有一个线程能抢占成功，那么其他的线程必须按序排好。同时也要考虑到多个节点往队伍中排会出现的并发问题
	2: 当上一个节点消费完成后，要将资源释放出来，然后要通知排在队伍中的第一个节点，通知他可以开始竞争资源了。
	
#### 所以我们可以带着这种思路来看AQS内部的实现，看是不是按照我们的设计思路来编写框架的。
## 正文
#### 接下来我们直接看到AQS的内部，大致浏览一下类的内部结构。
#### 首先可以看到，内部有一个叫做Node的内部类实现。这个类就是用于我们之前的设计中，当线程抢占资源失败时，排在队列中的每个单元。在Node的构造函数我们可以看到除了传入了当时的线程，还传入了一个叫mode的参数。
```
		Node(Thread thread, Node mode) {     // Used by addWaiter
			this.nextWaiter = mode;
   			this.thread = thread;
    	}
```
#### 结合了一些这个构造函数的调用链，可以看到这个mode有两种情况
```
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;
```
#### 这代表抢占资源的两种模式，一种是独占式，一种是共享式。
> 独占模式和共享模式。处于独占模式下时，其他线程试图获取该锁将无法取得成功。在共享模式下，多个线程获取某个锁可能（但不是一定）会获得成功

#### 这边引入了一个Node模式的概念。我们浏览过整个AQS的代码结构就可以发现，整份代码里就是有两种模式的不同入口，分别处理不同的模式。从名字也可以看出，比如下面的四个预留的抽象方法。

```
protected boolean tryAcquire(int arg)
protected boolean tryRelease(int arg)
protected int tryAcquireShared(int arg)
protected boolean tryReleaseShared(int arg)
```
#### 因为之前我们也说过AQS是一个核心的并发基础框架类，内部处理了很多关于抢占资源的调度方法，同时预留了一些可以处理实际逻辑的方法，供子类处理。以上四个方法就是预留的方法。从方法名字我们也可以知道，这四个方法里面属于两个不同的模式。在单个模式的两个方法中，也分别处理了试图去获取资源和试图去释放资源的实际方法。
#### 接下来我会尽量用图的方式将这两种模式给讲清楚。好了，先讲一下独占式的模式,这个模式的获取资源和释放资源的入口方法分别的Acquire和Release

## EXCLUSIVE模式
### 首先我们看一下独占模式的acquire方法
```
    public final void acquire(int arg) {
    	// 直接调用子类复写的方法，尝试去获取资源，
    	// 若成功，则直接返回
    	// 若失败，则加入内部维护的FIFO队列中
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
#### 简单看一下addWaiter方法内部
```
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 若当前队列中已经初始化了尾节点，则会先尝试一种快速入队的方式，即
        // 1：先获得当前队列中的尾节点，假设名字为t
        // 2: 将当前节点的先驱节点设置为t，即将节点排在了队列的最尾端
        // 3: 将队列的尾指针用CAS的方式，安全的指向当前节点。此处用CAS保证了线程安全
        // 4: 之后将原尾节点的后继指针指向了当前节点，从而完成了双向确认
        // 注意: 三个指针的操作中，之后尾指针的指向时用了CAS，其余两个都是普通的赋值操作，这边先埋个点，在之后会再讲到这个地方
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 若之前的方法成功，则直接返回
        // 否则就要进行比较复杂的方式
        enq(node);
        return node;
    }
    
    private Node enq(final Node node) {
    	// 整段代码由一个永真循环包围
        for (;;) {
            Node t = tail;
            // 若第一次调用，则要初始化队列的头尾指针
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
            	// 调用上段的逻辑，不断重试
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

####可以看到内部就是维护了一个FIFO的队列，队列中每个节点都维护了线程，和当前等待状态以及前驱和后继节点。新节点到来时，会用CAS+自旋的方法去将节点放在等待队列中。然后在acquireQueued方法中，我们也可以看到，每个在队列中的节点，都会不断根据前驱节点的状态的改变，不断的去尝试获取资源。
```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
        	// 标识线程是否中断
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // 判断先驱节点是否为队列首节点
                // 若是，则尝试去获取资源
                if (p == head && tryAcquire(arg)) {
                	//	若资源获取成功了，则将当前节点设为队列首节点
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 若还没轮到竞争资源，或者竞争资源失败
                // 每个节点都会判断自己以及前驱节点的状态是否满足当前节点是否可以park，这句话可能比较绕，可以直接先看看shouldParkAfterFailedAcquire这个方法里面做了什么
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
  	// 首先看到这个方法的返回值是一个布尔值，为true则标志当前节点可以park，会有其他节点在合适的时候将其唤醒
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    	// 先驱节点的状态
        int ws = pred.waitStatus;
		// 若前驱节点的状态是SIGNAL，也就是标志着当前节点在park前已经通知了前驱节点
		// 换句话说就是，前驱节点知道排在他后面的节点是park状态，那么在竞争完资源或者退出竞争之后，都会将其唤醒
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 若前驱节点的状态已经是取消状态，则应该跳过这个节点，向前找到未取消的节点作为新的前驱节点
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
             // 如果先驱节点已经取消竞争了，则直接跳过他，查找队列前面最近的没取消的作为新的先驱节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
             // 若先驱节点即没取消，又没Signal，则手动将其设置为Signal，表示当前节点已经安心park，等前驱节点资源释放后，需要将其唤醒
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 仅当先驱节点的状态是Signal的时候返回true，其他都为false
        return false;
    }

```
#### 整体逻辑就是在队列中等待的节点，会不断去尝试获得资源。因为是先入先出队列，所以节点都会根据排在前面的节点的状态，来判断是不是轮到自己了。shouldParkAfterFailedAcquire方法就是用于根据前驱节点的状态来确定自己是否可以park
#### 若shouldParkAfterFailedAcquire方法返回了true，就是需要去park当前线程，等待先驱节点完成之后唤醒。park部分的代码就是直接调用了LockSupport的park方法，等待唤醒。唤醒可能是被前驱节点唤醒，也有可能是线程中断，所以唤醒后要检查一下线程状态
```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
#### 简单总结一下acquireQueued方法的内部逻辑。就是排在队列中的每个节点，内部都有一个永真循环去不断判断是否可以去竞争资源，或者排在前面的节点是不是放弃竞争了，如果是，则向前找到最近的没放弃的节点作为先驱节点。当先驱节点的状态已经是Signal的时候，则park当前线程，等待唤醒。


### 再看一下独占模式的release方法
```
    public final boolean release(int arg) {
    	// 同样的，开始就去尝试释放资源，这个方法也是由子类去覆写的
        if (tryRelease(arg)) {
        	//	若释放成功，则得到等待队列中的头节点，如果节点存在，则唤醒头节点后的等待者，唤醒他开始竞争资源
            Node h = head;
            if (h != null && h.waitStatus != 0)
            	// 唤醒后继节点开始竞争资源
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
#### 和acquire方法一样，里面具体的try...()方法都是抽象方法，由继承的子类来覆写逻辑。当前节点释放资源成功后，要去主动提醒后续节点。这部分的逻辑在unparkSuccessor方法中实现
```
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
         // 从队列的末尾向前找到后继节点中，没有取消的节点，去唤醒。
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 找到有效的后继节点，尝试去唤醒
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
#### 这边逻辑大部分都是易懂的，无非就是释放资源后，通知队列中排在后面的节点开始竞争。但是为什么倒序？ 
#### 因为考虑enq方法的时候，后继节点可能为null。也就是我们之前说的入队的时候，埋下的坑
```
        if (pred != null) {
            node.prev = pred;                    1
            if (compareAndSetTail(pred, node)) { 2
                pred.next = node;                3
                return node;                     4
            }
        }
```
#### 观察这段代码，其中只有在2的时候，是用CAS，保证尾指针指向了新加入的节点。而且在CAS之前，已经将当前节点的前驱节点设置为了原来的尾指针。CAS完成后，进行3。这边是一个简单的赋值语句。所以考虑这么一种情况
- 当2步骤完成时，由于是多线程。此时如果调度到了别的线程的release方法。若按照正常的正序查找后继结点。可以发现由于3步骤还没执行，原尾节点是没有后继节点的。那么就会认为队列已经空了，这显然是不对的。
- 但是由于尾指针的指向时CAS操作的，所以由尾指针作为入口，肯定是没问题的
- 倒序查找还跟CANCEL操作的时候有关，这部分可以直接看我文章下面的链接。写的非常好，我写不出那么好，就不班门弄斧了
#### 这里注意几点细节。

- 当前节点释放资源成功后，要去唤醒队伍中排在后面的人去消费。按常理来说，应该就是排在后面的那个节点，但是由于每个节点都有可能取消竞争状态，所以要跳过这种节点，以及跳过为空的节点
- 查找的时候要从队伍的末尾往前查找

#### 唤醒成功后，就完成了释放资源的过程。之前park住的线程会被唤醒，继续他的竞争资源的过程


### 共享模式的acquire方法
#### 上面说完了独占模式下，资源竞争的问题，现在我们再看看另一种共享模式下有什么不一样。
#### 之前也说过，共享模式就是允许多个线程同时占有资源的情形。那这个acquireShared方法是怎么实现的呢？
```
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
#### 回想一下之前说的独占模式的acquire方法。可以很明显的看到几点不同
- 独占模式下try...()方法返回的是一个布尔值，直接确定是否抢占到了资源。而在共享模式下，返回的是一个int值，为负数的时候则表明抢占失败
- 共享模式这边少了一句 selfInterrupt()

#### 和独占模式一样，如果尝试获取失败，会在队列中新建一个节点，持续等待机会去抢占
```
	private void doAcquireShared(int arg) {
		// 注意这里的Node的模式已经是SHARED了
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                	// 同样的，注意这个返回是一个int值
                    int r = tryAcquireShared(arg);
                   // 返回值是正数，代表获取成功
                    if (r >= 0) {
                    	// 这是共享模式独有的Node状态
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
#### 大部分逻辑和独占模式的时候差不多，独有的代码部分就是返回值为int以及setHeadAndPropagate这个方法，我们再看看这个方法里面做了什么
```
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        // 当前节点抢占资源的返回值，标记之后的运行逻辑
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```


## 超链接：
[关于为何倒序查找的Cancel方法论证](http://www.ideabuffer.cn/2017/03/15/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3AbstractQueuedSynchronizer%EF%BC%88%E4%B8%80%EF%BC%89/)