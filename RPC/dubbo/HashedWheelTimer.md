#### 时间轮算法 HashedWheelTimer
#### 起因
由于netty动辄管理100w+的连接，每一个连接都会有很多超时任务。比如发送超时、心跳检测间隔等，如果每一个定时任务都启动一个Timer,不仅低效，而且会消耗大量的资源。
后来被广泛应用于dubbo，以及Kafka等其他系统性演示任务。
##### 1：为什么采用时间轮算法来处理调度
大量的调度任务如果每一个都使用自己的调度器来管理任务的生命周期的话，浪费cpu的资源并且很低效。

时间轮是一种高效来利用线程资源来进行批量化调度的一种调度模型。把大批量的调度任务全部都绑定到同一个的调度器上面，使用这一个调度器来进行所有任务的管理（manager），触发（trigger）以及运行（runnable）。能够高效的管理各种延时任务，周期任务，通知任务等等。

缺点，时间轮调度器的时间精度可能不是很高，对于精度要求特别高的调度任务可能不太适合。因为时间轮算法的精度取决于，时间段“指针”单元的最小粒度大小，比如时间轮的格子是一秒跳一次，那么调度精度小于一秒的任务就无法被时间轮所调度。而且时间轮算法没有做宕机备份，因此无法再宕机之后恢复任务重新调度。
#### 有赞在时间轮方面的一个应用以及对比（）
时间轮
经常会面临一些业务定时超时的需求，用例子来说明吧。

功能需求：服务器需要维护来自大量客户端的TCP连接（假设单机服务器需要支持的最大TCP连接数在10W级别），如果某连接上60s内没有数据到达，就认为相应的客户端下线。

先介绍一下两种容易想到的解决方案,

方案a 轮询扫描

处理过程为：

维护一个map<client_id, last_update_time > 记录客户端最近一次的请求时间；
当client_id对应连接有数据到达时，更新last_update_time；
启动一个定时器，轮询扫描map 中client_id 对应的last_update_time，若超过 60s，则认为对应的客户端下线。
轮询扫描，只启动一个定时器，但轮询效率低，特别是服务器维护的连接数很大时，部分连接超时事件得不到及时处理。

方案b 多定时器触发

处理过程为：

维护一个map<client_id, last_update_time > 记录客户端最近一次的请求时间；
当某client_id 对应连接有数据到达时，更新last_update_time，同时为client_id启用一个定时器，60s后触发;
当client_id对应的定时器触发后，查看map中client_id对应的last_update_time是否超过60s，若超时则认为对应客户端下线。
多定时器触发，每次请求都要启动一个定时器，可以想象，消息请求非常频繁是，定时器的数量将会很庞大，消耗大量的系统资源。

方案c 时间轮方案

下面介绍一下利用时间轮的方式实现的一种高效、能批量的处理方案，先说一下需要的数据结构：

创建0~60的数据，构成环形队列time_wheel，current_index维护环形队列的当前游标，如图19所示；
数组元素是slot 结构，slot是一个set<client_id>，构成任务集；
维护一个map<client_id,index>，记录client_id 落在哪个slot上。


                     图19 时间轮环形队列示意图
执行过程为：

启用一个定时器，运行间隔1s，更新current_index，指向环形队列下一个元素，0->1->2->3...->58->59->60...0；
连接上数据到达时，从map中获取client_id所在的slot，在slot的set中删除该client_id；
将client_id加入到current_index - 1锁标记的slot中；
更新map中client_id 为current_id-1 。
与a、b两种方案相比，方案c具有如下优势：

只需要一个定时器，运行间隔1s，CPU消耗非常少；
current_index 所标记的slot中的set不为空时，set中的所有client_id对应的客户端均认为下线，即批量超时。
上面描述的时间轮处理方式会存在1s以内的误差，若考虑实时性，可以提高定时器的运行间隔，另外该方案可以根据实际业务需求扩展到应用中。我们对Swoole的修改中，包括对定时器进行了重构，其中超时定时器采用的就是如上所描述的时间轮方案，并且精度可控。


#### 2：原理

时间轮其实就是一种环形的数据结构，可以想象成时钟，分成很多格子，一个格子代码一段时间（这个时间越短，Timer的精度越高）。并用一个链表报错在该格子上的到期任务，同时一个指针随着时间一格一格转动，并执行相应格子中的到期任务。任务通过取摸决定放入那个格子。

####3：添加延时任务的几种情况：

##### 一、时间到期

1）比如说有一个任务要在 16:29:07 执行，从秒级时间轮中来看，当我们的当前时间走到16:29:06的时候，则表示这个任务已经过期了。因为它的delayMs = 1000ms，小于了我们的秒级时间轮的tickMs(1000ms)。
比如说有一个任务要在 16:41:25 执行，从分级时间轮中来看，当我们的当前时间走到 16:41的时候（ 分级时间轮没有秒针！它的最小精度是分钟（一定要理解这一点） ），则表示这个任务已经到期，因为它的delayMs = 25000ms，小于了我们的分级时间轮的tickMs（60000ms）。
##### 二、时间未到期，且delayMs小于interval。

##### 对于秒级时间轮来说，就是延迟时间小于60s，那么肯定能找到一个秒钟槽扔进去。

##### 三、时间未到期，且delayMs大于interval。

对于妙级时间轮来说，就是延迟时间大于等于60s，这时候就需要借助上层时间轮的力量了，很简单的代码实现，就是拿到上层时间轮，然后类似递归一样，把它扔进去。

比如说一个有一个延时为一年后的定时任务，就会在这个递归中不断创建更上层的时间轮，直到找到满足delayMs小于interval的那个时间轮。

这里为了不把代码写的那么复杂，我们每一层时间轮的刻度都一样，也就是秒级时间轮表示60秒，上面则表示60分钟，再上面则表示60小时，再上层则表示60个60小时，再上层则表示60个60个60小时 = 216000小时。

也就是如果将最底层时间轮的tickMs（精度）设置为1000ms。wheelSize设置为60。 那么只需要5层时间轮，可表示的时间跨度已经长达24年（216000小时） 。

当然我们的时间轮还需要一个指针的推进机制，总不能让时间永远停留在当前吧？推进的时候，同时类似递归，去推进一下上一层的时间轮。

注意：要强调一点的是，我们这个时间轮更像是电子表，它不存在时间的中间状态，也就是精度这个概念一定要理解好。比如说，对于秒级时间轮来说，它的精度只能保证到1秒，小于1秒的，都会当成是已到期

对于分级时间轮来说，它的精度只能保证到1分，小于1分的，都会当成是已到期

##### 高层时间轮与底层时间轮的冲突

 分级时间轮，精度只有分钟级，总不能延迟1秒的定时任务和延迟59秒的定时任务同时执行吧？

 只需再入时间轮即可。比如说，对于分钟级时间轮来说，delayMs为1秒和delayMs为59秒的都已经过期，我们将其取出，再扔进底层的时间轮不就可以了？

1秒的会被扔到秒级时间轮的下一个执行槽中，而59秒的会被扔到秒级时间轮的后59个时间槽中。



##### 如何知道一个任务已经过期？

记得我们将任务存储在槽中嘛？比如说秒级时间轮中，有60个槽，那么一共有60个槽。如果时间轮共有两层，也仅仅只有120个槽。我们只需将槽扔进一个delayedQueue之中即可。

我们轮询地从delayedQueue取出已经过期的槽即可。（前面的所有代码，为了简单说明，并没有引入这个DelayQueue的概念，所以不用去上面翻了，并没有。博主觉得... 已经看到这里了，应该很明白这个DelayQueue的意义了。 ）

其实简单来说，实际上定时任务单单使用DelayQueue来实现，也是可以的，但是一旦任务的数量多了起来，达到了百万级，千万级，针对这个delayQueue的增删，将非常的慢。

** 一、面向槽的delayQueue**

而对于时间轮来说，它只需要往delayQueue里面扔各种槽即可，比如我们的定时任务长短不一，最长的跨度到了24年，这个delayQueue也仅仅只有300个元素。

** 二、处理过期的槽**

而这个槽到期后，也就是被我们从delayQueue中poll出来后，我们只需要将槽中的所有任务循环一次，重新加到新的槽中（添加失败则直接执行）即可。

#### 源码解读
##### 构造方法
```java
public HashedWheelTimer(
          ThreadFactory threadFactory, // 用来创建worker线程
          long tickDuration, // tick的时长，也就是指针多久转一格
          TimeUnit unit, // tickDuration的时间单位
          int ticksPerWheel, // 一圈有几格
          boolean leakDetection // 是否开启内存泄露检测
          ) {
      // 一些参数校验
      if (threadFactory == null) {
          throw new NullPointerException("threadFactory");
      }
      if (unit == null) {
          throw new NullPointerException("unit");
      }
      if (tickDuration <= 0) {
          throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
      }
      if (ticksPerWheel <= 0) {
          throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
      }
      // 创建时间轮基本的数据结构，一个数组。长度为不小于ticksPerWheel的最小2的n次方
      wheel = createWheel(ticksPerWheel);
      // 这是一个标示符，用来快速计算任务应该呆的格子。
      // 我们知道，给定一个deadline的定时任务，其应该呆的格子=deadline%wheel.length.但是%操作是个相对耗时的操作，所以使用一种变通的位运算代替：
      // 因为一圈的长度为2的n次方，mask = 2^n-1后低位将全部是1，然后deadline&mast == deadline%wheel.length
      // java中的HashMap在进行hash之后，进行index的hash寻址寻址的算法也是和这个一样的
      mask = wheel.length - 1;
      // 转换成纳秒处理
      this.tickDuration = unit.toNanos(tickDuration);
      // 校验是否存在溢出。即指针转动的时间间隔不能太长而导致tickDuration*wheel.length>Long.MAX_VALUE
      if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
          throw new IllegalArgumentException(String.format(
                  "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                  tickDuration, Long.MAX_VALUE / wheel.length));
      }
      // 创建worker线程
      workerThread = threadFactory.newThread(worker);
// 这里默认是启动内存泄露检测：当HashedWheelTimer实例超过当前cpu可用核数*4的时候，将发出警告
      leak = leakDetection || !workerThread.isDaemon() ? leakDetector.open(this) : null;
    //任务的超时等待时间，如果设置了超时了之后会抛出异常  
    this.maxPendingTimeouts = maxPendingTimeouts;
    //当HashedWheelTimer实例超过当前cpu可用核数最大的实例数为64个，也就是一个jvm最多只能有64个时间轮对象实例，
因为时间轮是一个非常耗费资源的结构所以实例数目不能太高
      if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        //超过会报警实例过多
        //  String resourceType = simpleClassName(HashedWheelTimer.class);
        //logger.error("You are creating too many " + resourceType + " instances. " +
        //        resourceType + " is a shared resource that must be reused across the  //JVM," +
  //              "so that only a few instances are created.");
        }
  }
 ``` 
##### HashedWheelTimer源码之启动、停止与添加任务
```java
// 启动时间轮。这个方法其实不需要显示的主动调用，因为在添加定时任务（newTimeout()方法）的时候会自动调用此方法。
// 这个是合理的设计，因为如果时间轮里根本没有定时任务，启动时间轮也是空耗资源，因此使用的类似于hashMap的懒加载的形式，
//只有当有任务提交之后才会启动这个调度器
//AtomicIntegerFieldUpdater是JUC里面的类，原理是利用安全的反射进行原子操作，来获取实例的本身的属性。有比AtomicInteger更好的性能和更低得内存占用
public void start() {
    // 判断当前时间轮的状态，如果是初始化，则启动worker线程，启动整个时间轮；如果已经启动则略过；如果是已经停止，则报错
    // 这里是一个Lock Free的设计。因为可能有多个线程调用启动方法，这里使用AtomicIntegerFieldUpdater原子的更新时间轮的状态
//，因此使用这个方法来获取实例的允许状态，防止重入
    switch (WORKER_STATE_UPDATER.get(this)) {
        case WORKER_STATE_INIT:
        //使用cas来获取启动调度的权力，只有竞争到的线程允许来进行实例启动
            if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                workerThread.start();
            }
            break;
        case WORKER_STATE_STARTED:
            break;
        case WORKER_STATE_SHUTDOWN:
            throw new IllegalStateException("cannot be started once stopped");
        default:
            throw new Error("Invalid WorkerState");
    }

    // 等待worker线程初始化时间轮的启动时间
    while (startTime == 0) {
        try {
          //这里使用countDownLauch来确保调度的线程已经被启动
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}

```
stop()停止时间轮的方法：

```java

public Set<Timeout> stop() {
    // worker线程不能停止时间轮，也就是加入的定时任务，不能调用这个方法。
    // 不然会有恶意的定时任务调用这个方法而造成大量定时任务失效
    if (Thread.currentThread() == workerThread) {
        throw new IllegalStateException(
                HashedWheelTimer.class.getSimpleName() +
                        ".stop() cannot be called from " +
                        TimerTask.class.getSimpleName());
    }
    // 尝试CAS替换当前状态为“停止：2”。如果失败，则当前时间轮的状态只能是“初始化：0”或者“停止：2”。
//直接将当前状态设置为“停止：2“
    if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
        // workerState can be 0 or 2 at this moment - let it always be 2.
        WORKER_STATE_UPDATER.set(this, WORKER_STATE_SHUTDOWN);

        if (leak != null) {
            leak.close();
        }

        return Collections.emptySet();
    }

    // 终端worker线程，尝试把正在进行任务的线程中断掉,如果某些任务正在执行则会，
//抛出interrupt异常，并且任务会尝试立即中断
    boolean interrupted = false;
    while (workerThread.isAlive()) {
        workerThread.interrupt();
        try {
          //当前前程会等待stop的结果
            workerThread.join(100);
        } catch (InterruptedException ignored) {
            interrupted = true;
        }
    }

    // 从中断中恢复
    if (interrupted) {
          // 如果中断掉了所有工作的线程，那么当前关闭时间轮调度的线程会在随后关闭
        Thread.currentThread().interrupt();
    }

    if (leak != null) {
        leak.close();
    }
    // 返回未处理的任务
    return worker.unprocessedTimeouts();
}
```
添加定时任务
```java

public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    // 参数校验
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }
    // 如果时间轮没有启动，则启动
    start();

    // Add the timeout to the timeout queue which will be processed on the next tick.
    // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
    // 计算任务的deadline
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;
    // 这里定时任务不是直接加到对应的格子中，而是先加入到一个队列里，然后等到下一个tick的时候，会从队列里取出最多100000个任务加入到指定的格子中
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    timeouts.add(timeout);
    return timeout;
}
```
#### 全部源码 
```java
/*
 * Copyright 2012 The Netty Project
 *
 * The Netty Project licenses this file to you under the Apache License,
 * version 2.0 (the "License"); you may not use this file except in compliance
 * with the License. You may obtain a copy of the License at:
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 * WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 * License for the specific language governing permissions and limitations
 * under the License.
 */

package org.apache.dubbo.common.timer;

import org.apache.dubbo.common.logger.Logger;
import org.apache.dubbo.common.logger.LoggerFactory;
import org.apache.dubbo.common.utils.ClassHelper;

import java.util.Collections;
import java.util.HashSet;
import java.util.Locale;
import java.util.Queue;
import java.util.Set;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Executors;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;
import java.util.concurrent.atomic.AtomicLong;

/**
 * A {@link Timer} optimized for approximated I/O timeout scheduling.
 *
 * <h3>Tick Duration</h3>
 * <p>
 * As described with 'approximated', this timer does not execute the scheduled
 * {@link TimerTask} on time.  {@link HashedWheelTimer}, on every tick, will
 * check if there are any {@link TimerTask}s behind the schedule and execute
 * them.
 * <p>
 * You can increase or decrease the accuracy of the execution timing by
 * specifying smaller or larger tick duration in the constructor.  In most
 * network applications, I/O timeout does not need to be accurate.  Therefore,
 * the default tick duration is 100 milliseconds and you will not need to try
 * different configurations in most cases.
 *
 * <h3>Ticks per Wheel (Wheel Size)</h3>
 * <p>
 * {@link HashedWheelTimer} maintains a data structure called 'wheel'.
 * To put simply, a wheel is a hash table of {@link TimerTask}s whose hash
 * function is 'dead line of the task'.  The default number of ticks per wheel
 * (i.e. the size of the wheel) is 512.  You could specify a larger value
 * if you are going to schedule a lot of timeouts.
 *
 * <h3>Do not create many instances.</h3>
 * <p>
 * {@link HashedWheelTimer} creates a new thread whenever it is instantiated and
 * started.  Therefore, you should make sure to create only one instance and
 * share it across your application.  One of the common mistakes, that makes
 * your application unresponsive, is to create a new instance for every connection.
 *
 * <h3>Implementation Details</h3>
 * <p>
 * {@link HashedWheelTimer} is based on
 * <a href="http://cseweb.ucsd.edu/users/varghese/">George Varghese</a> and
 * Tony Lauck's paper,
 * <a href="http://cseweb.ucsd.edu/users/varghese/PAPERS/twheel.ps.Z">'Hashed
 * and Hierarchical Timing Wheels: data structures to efficiently implement a
 * timer facility'</a>.  More comprehensive slides are located
 * <a href="http://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt">here</a>.
 */
public class HashedWheelTimer implements Timer {

    /**
     * HashedWheelTimer的注释特别说明了不要创建多个HashedWheelTimer对象, HashedWheelTimer是用来管理大量的定时任务的,
     * 但这些任务要放在同一个Timer对象里面管理
     */
    public static final String NAME = "hased";

    private static final Logger logger = LoggerFactory.getLogger(HashedWheelTimer.class);

    private static final AtomicInteger INSTANCE_COUNTER = new AtomicInteger();
    private static final AtomicBoolean WARNED_TOO_MANY_INSTANCES = new AtomicBoolean();
    private static final int INSTANCE_COUNT_LIMIT = 64;
    //采用cas的方式更新定时任务状态 AtomicIntegerFieldUpdater是JUC里面的类，原理是利用安全的反射进行原子操作，来获取实例的本身的属性。有比AtomicInteger更好的性能和更低得内存占用。
    private static final AtomicIntegerFieldUpdater<HashedWheelTimer> WORKER_STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimer.class, "workerState");

    private final Worker worker = new Worker();
    private final Thread workerThread;
    //定时任务的三种状态
    private static final int WORKER_STATE_INIT = 0;
    private static final int WORKER_STATE_STARTED = 1;
    private static final int WORKER_STATE_SHUTDOWN = 2;

    /**
     * 0 - init, 1 - started, 2 - shut down
     */
    @SuppressWarnings({"unused", "FieldMayBeFinal"})
    private volatile int workerState;
    //间隔多久走到下一槽（相当于时钟走一格）
    private final long tickDuration;
    private final HashedWheelBucket[] wheel;
    private final int mask;
    private final CountDownLatch startTimeInitialized = new CountDownLatch(1);
    private final Queue<HashedWheelTimeout> timeouts = new ArrayBlockingQueue<HashedWheelTimeout>(1024);
    private final Queue<HashedWheelTimeout> cancelledTimeouts = new ArrayBlockingQueue<HashedWheelTimeout>(1024);
    private final AtomicLong pendingTimeouts = new AtomicLong(0);
    private final long maxPendingTimeouts;

    private volatile long startTime;

    /**
     * Creates a new timer with the default thread factory
     * ({@link Executors#defaultThreadFactory()}), default tick duration, and
     * default number of ticks per wheel.
     */
    public HashedWheelTimer() {
        this(Executors.defaultThreadFactory());
    }

    /**
     * Creates a new timer with the default thread factory
     * ({@link Executors#defaultThreadFactory()}) and default number of ticks
     * per wheel.
     *
     * @param tickDuration the duration between tick
     * @param unit         the time unit of the {@code tickDuration}
     * @throws NullPointerException     if {@code unit} is {@code null}
     * @throws IllegalArgumentException if {@code tickDuration} is &lt;= 0
     */
    public HashedWheelTimer(long tickDuration, TimeUnit unit) {
        this(Executors.defaultThreadFactory(), tickDuration, unit);
    }

    /**
     * Creates a new timer with the default thread factory
     * ({@link Executors#defaultThreadFactory()}).
     *
     * @param tickDuration  the duration between tick
     * @param unit          the time unit of the {@code tickDuration}
     * @param ticksPerWheel the size of the wheel
     * @throws NullPointerException     if {@code unit} is {@code null}
     * @throws IllegalArgumentException if either of {@code tickDuration} and {@code ticksPerWheel} is &lt;= 0
     */
    public HashedWheelTimer(long tickDuration, TimeUnit unit, int ticksPerWheel) {
        this(Executors.defaultThreadFactory(), tickDuration, unit, ticksPerWheel);
    }

    /**
     * Creates a new timer with the default tick duration and default number of
     * ticks per wheel.
     *
     * @param threadFactory a {@link ThreadFactory} that creates a
     *                      background {@link Thread} which is dedicated to
     *                      {@link TimerTask} execution.
     * @throws NullPointerException if {@code threadFactory} is {@code null}
     */
    public HashedWheelTimer(ThreadFactory threadFactory) {
        this(threadFactory, 100, TimeUnit.MILLISECONDS);
    }

    /**
     * Creates a new timer with the default number of ticks per wheel.
     *
     * @param threadFactory a {@link ThreadFactory} that creates a
     *                      background {@link Thread} which is dedicated to
     *                      {@link TimerTask} execution.
     * @param tickDuration  the duration between tick
     * @param unit          the time unit of the {@code tickDuration}
     * @throws NullPointerException     if either of {@code threadFactory} and {@code unit} is {@code null}
     * @throws IllegalArgumentException if {@code tickDuration} is &lt;= 0
     */
    public HashedWheelTimer(
            ThreadFactory threadFactory, long tickDuration, TimeUnit unit) {
        this(threadFactory, tickDuration, unit, 512);
    }

    /**
     * Creates a new timer.
     *
     * @param threadFactory a {@link ThreadFactory} that creates a
     *                      background {@link Thread} which is dedicated to
     *                      {@link TimerTask} execution.
     * @param tickDuration  the duration between tick
     * @param unit          the time unit of the {@code tickDuration}
     * @param ticksPerWheel the size of the wheel
     * @throws NullPointerException     if either of {@code threadFactory} and {@code unit} is {@code null}
     * @throws IllegalArgumentException if either of {@code tickDuration} and {@code ticksPerWheel} is &lt;= 0
     */
    public HashedWheelTimer(
            ThreadFactory threadFactory,
            long tickDuration, TimeUnit unit, int ticksPerWheel) {
        this(threadFactory, tickDuration, unit, ticksPerWheel, -1);
    }

    /**
     * Creates a new timer.
     *
     * @param threadFactory      a {@link ThreadFactory} that creates a
     *                           background {@link Thread} which is dedicated to
     *                           {@link TimerTask} execution.
     * @param tickDuration       the duration between tick
     * @param unit               the time unit of the {@code tickDuration}
     * @param ticksPerWheel      the size of the wheel
     * @param maxPendingTimeouts The maximum number of pending timeouts after which call to
     *                           {@code newTimeout} will result in
     *                           {@link java.util.concurrent.RejectedExecutionException}
     *                           being thrown. No maximum pending timeouts limit is assumed if
     *                           this value is 0 or negative.
     * @throws NullPointerException     if either of {@code threadFactory} and {@code unit} is {@code null}
     * @throws IllegalArgumentException if either of {@code tickDuration} and {@code ticksPerWheel} is &lt;= 0
     */
    public HashedWheelTimer(
            ThreadFactory threadFactory,
            long tickDuration, TimeUnit unit, int ticksPerWheel,
            long maxPendingTimeouts) {

        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }
        if (tickDuration <= 0) {
            throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
        }
        if (ticksPerWheel <= 0) {
            throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
        }

        // Normalize ticksPerWheel to power of two and initialize the wheel.
        //创建时间轮的基本数据结构 一个数组 长度会自动转换为2的N次方  一般如果我们hash取模定位的话，用%  但这个没有二进制运算高效
        wheel = createWheel(ticksPerWheel);
        //用来快速计算我们的任务应该处于时间槽哪一个位置  也就是进行&操作
        mask = wheel.length - 1;

        // Convert tickDuration to nanos.时间转换成纳秒
        this.tickDuration = unit.toNanos(tickDuration);

        // Prevent overflow.
        if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
            throw new IllegalArgumentException(String.format(
                    "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                    tickDuration, Long.MAX_VALUE / wheel.length));
        }
        //创建work线程
        workerThread = threadFactory.newThread(worker);
        //任务的超时等待时间如果有设置超时 那么超时之后就会抛异常
        this.maxPendingTimeouts = maxPendingTimeouts;
        //这玩意比较耗资源，所以会控制实例化数目
        if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
                WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        }
    }

    @Override
    protected void finalize() throws Throwable {
        try {
            super.finalize();
        } finally {
            // This object is going to be GCed and it is assumed the ship has sailed to do a proper shutdown. If
            // we have not yet shutdown then we want to make sure we decrement the active instance count.
            if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
                INSTANCE_COUNTER.decrementAndGet();
            }
        }
    }

    //创建槽位
    private static HashedWheelBucket[] createWheel(int ticksPerWheel) {
        if (ticksPerWheel <= 0) {
            throw new IllegalArgumentException(
                    "ticksPerWheel must be greater than 0: " + ticksPerWheel);
        }
        if (ticksPerWheel > 1073741824) {
            throw new IllegalArgumentException(
                    "ticksPerWheel may not be greater than 2^30: " + ticksPerWheel);
        }

        ticksPerWheel = normalizeTicksPerWheel(ticksPerWheel);
        HashedWheelBucket[] wheel = new HashedWheelBucket[ticksPerWheel];
        for (int i = 0; i < wheel.length; i++) {
            wheel[i] = new HashedWheelBucket();
        }
        return wheel;
    }

    //计算槽位个数
    private static int normalizeTicksPerWheel(int ticksPerWheel) {
        int normalizedTicksPerWheel = 1;
        //所以最终结果一定是2的n次方
        while (normalizedTicksPerWheel < ticksPerWheel) {
            normalizedTicksPerWheel <<= 1;
        }
        return normalizedTicksPerWheel;
    }

    /**
     * Starts the background thread explicitly.  The background thread will
     * start automatically on demand even if you did not call this method.
     *
     * @throws IllegalStateException if this timer has been
     *                               {@linkplain #stop() stopped} already
     */
    //不需要你主动调用，当有任务添加进来的的时候他就会跑
    public void start() {
        // 判断当前时间轮的状态，如果是初始化，则启动worker线程，启动整个时间轮；如果已经启动则略过；如果是已经停止，则报错
        // 这里是一个Lock Free的设计。因为可能有多个线程调用启动方法，这里使用AtomicIntegerFieldUpdater原子的更新时间轮的状态
        //因此使用这个方法来获取实例的允许状态，防止重入
        switch (WORKER_STATE_UPDATER.get(this)) {
            case WORKER_STATE_INIT:
                //cas锁 保证只有一个能够启动
                if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                    workerThread.start();
                }
                break;
            case WORKER_STATE_STARTED:
                break;
            case WORKER_STATE_SHUTDOWN:
                throw new IllegalStateException("cannot be started once stopped");
            default:
                throw new Error("Invalid WorkerState");
        }

        // Wait until the startTime is initialized by the worker.
        //等待work线程初始化时间轮的启动时间
        while (startTime == 0) {
            try {
                //这里使用countDownLauch来确保调度的线程已经被启动
                startTimeInitialized.await();
            } catch (InterruptedException ignore) {
                // Ignore - it will be ready very soon.
            }
        }
    }

    @Override
    public Set<Timeout> stop() {
        // worker线程不能停止时间轮，也就是加入的定时任务，不能调用这个方法。
        // 不然会有恶意的定时任务调用这个方法而造成大量定时任务失效
        if (Thread.currentThread() == workerThread) {
            throw new IllegalStateException(
                    HashedWheelTimer.class.getSimpleName() +
                            ".stop() cannot be called from " +
                            TimerTask.class.getSimpleName());
        }
        // 尝试CAS替换当前状态为“停止：2”。如果失败，则当前时间轮的状态只能是“初始化：0”或者“停止：2”。
        //直接将当前状态设置为“停止：2“
        if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
            // workerState can be 0 or 2 at this moment - let it always be 2.
            if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
                INSTANCE_COUNTER.decrementAndGet();
            }

            return Collections.emptySet();
        }

        try {
            // 终端worker线程，尝试把正在进行任务的线程中断掉,如果某些任务正在执行则会，
           //抛出interrupt异常，并且任务会尝试立即中断
            boolean interrupted = false;
            while (workerThread.isAlive()) {
                workerThread.interrupt();
                try {
                    //当前前程会等待stop的结果
                    workerThread.join(100);
                } catch (InterruptedException ignored) {
                    interrupted = true;
                }
            }

            // 从中断中恢复

            if (interrupted) {
                // 如果中断掉了所有工作的线程，那么当前关闭时间轮调度的线程会在随后关闭
                Thread.currentThread().interrupt();
            }
        } finally {
            INSTANCE_COUNTER.decrementAndGet();
        }
        //返回未处理的任务
        return worker.unprocessedTimeouts();
    }

    @Override
    public boolean isStop() {
        return WORKER_STATE_SHUTDOWN == WORKER_STATE_UPDATER.get(this);
    }
    //添加定时任务
    @Override
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();

        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                    + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                    + "timeouts (" + maxPendingTimeouts + ")");
        }
        //如果没有启动，则启动
        start();

        // Add the timeout to the timeout queue which will be processed on the next tick.
        // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

        // Guard against overflow.
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
        //// 这里定时任务不是直接加到对应的格子中，而是先加入到一个队列里，然后等到下一个tick的时候，会从队列里取出最多100000个任务加入到指定的格子中
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        timeouts.add(timeout);
        return timeout;
    }

    /**
     * Returns the number of pending timeouts of this {@link Timer}.
     */
    public long pendingTimeouts() {
        return pendingTimeouts.get();
    }

    private static void reportTooManyInstances() {
        String resourceType = ClassHelper.simpleClassName(HashedWheelTimer.class);
        logger.error("You are creating too many " + resourceType + " instances. " +
                resourceType + " is a shared resource that must be reused across the JVM," +
                "so that only a few instances are created.");
    }

    private final class Worker implements Runnable {
        private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

        private long tick;

        @Override
        public void run() {
            // Initialize the startTime.
            // 初始化startTime.只有所有任务的的deadline都是想对于这个时间点
            startTime = System.nanoTime();
            // 由于System.nanoTime()可能返回0，甚至负数。并且0是一个标示符，用来判断startTime是否被初始化，所以当startTime=0的时候，重新赋值为1
            if (startTime == 0) {
                // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
                startTime = 1;
            }

            // Notify the other threads waiting for the initialization at start().
            //唤醒阻塞在start的线程
            startTimeInitialized.countDown();

            do {
                // 计算需要sleep的时间
                final long deadline = waitForNextTick();
                if (deadline > 0) {
                    int idx = (int) (tick & mask);
                    processCancelledTasks();
                    HashedWheelBucket bucket =
                            wheel[idx];
                    transferTimeoutsToBuckets();
                    bucket.expireTimeouts(deadline);
                    tick++;
                }
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

            // Fill the unprocessedTimeouts so we can return them from stop() method.
            //填充未处理的超时
            for (HashedWheelBucket bucket : wheel) {
                bucket.clearTimeouts(unprocessedTimeouts);
            }
            for (; ; ) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    break;
                }
                if (!timeout.isCancelled()) {
                    unprocessedTimeouts.add(timeout);
                }
            }
            processCancelledTasks();
        }
        // 将newTimeout()方法中加入到待处理定时任务队列中的任务加入到指定的格子中
        private void transferTimeoutsToBuckets() {
            // transfer only max. 100000 timeouts per tick to prevent a thread to stale the workerThread when it just
            // adds new timeouts in a loop.
            // 每次tick只处理10w个任务，以免阻塞worker线程
            for (int i = 0; i < 100000; i++) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    // all processed
                    break;
                }
                //已经被取消了；
                if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                    // Was cancelled in the meantime.
                    continue;
                }
                // 计算任务需要经过多少个tick
                long calculated = timeout.deadline / tickDuration;
                // 计算任务的轮数
                timeout.remainingRounds = (calculated - tick) / wheel.length;

                // Ensure we don't schedule for past.
                //如果任务在timeouts队列里面放久了, 以至于已经过了执行时间, 这个时候就使用当前tick, 也就是放到当前bucket, 此方法调用完后就会被执行.
                final long ticks = Math.max(calculated, tick);
                int stopIndex = (int) (ticks & mask);
                // 将任务加入到响应的格子中
                HashedWheelBucket bucket = wheel[stopIndex];
                bucket.addTimeout(timeout);
            }
        }
        // 将取消的任务取出，并从格子中移除
        private void processCancelledTasks() {
            for (; ; ) {
                HashedWheelTimeout timeout = cancelledTimeouts.poll();
                if (timeout == null) {
                    // all processed
                    break;
                }
                try {
                    timeout.remove();
                } catch (Throwable t) {
                    if (logger.isWarnEnabled()) {
                        logger.warn("An exception was thrown while process a cancellation task", t);
                    }
                }
            }
        }

        /**
         * calculate goal nanoTime from startTime and current tick number,
         * then wait until that goal has been reached.
         *
         * @return Long.MIN_VALUE if received a shutdown request,
         * current time otherwise (with Long.MIN_VALUE changed by +1)
         */
        //sleep, 直到下次tick到来, 然后返回该次tick和启动时间之间的时长
        private long waitForNextTick() {
            long deadline = tickDuration * (tick + 1);

            for (; ; ) {
                // 计算需要sleep的时间, 之所以加999999后再除10000000,前面是1所以这里需要减去1，才能计算准确，还有通过这里可以看到，
//其实线程是以睡眠一定的时候再来执行下一个ticket的任务的，
//这样如果ticket的间隔设置的太小的话，系统会频繁的睡眠然后启动，
//其实感觉影响部分的性能，所以为了更好的利用系统资源步长可以稍微设置大点
                final long currentTime = System.nanoTime() - startTime;
                long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

                if (sleepTimeMs <= 0) {
                    if (currentTime == Long.MIN_VALUE) {
                        return -Long.MAX_VALUE;
                    } else {
                        return currentTime;
                    }
                }
                if (isWindows()) {
                    // 这里是因为windows平台的定时调度最小单位为10ms，如果不是10ms的倍数，可能会引起sleep时间不准确
                    sleepTimeMs = sleepTimeMs / 10 * 10;
                }

                try {
                    Thread.sleep(sleepTimeMs);
                } catch (InterruptedException ignored) {
                    // 调用HashedWheelTimer.stop()时优雅退出
                    if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                        return Long.MIN_VALUE;
                    }
                }
            }
        }

        Set<Timeout> unprocessedTimeouts() {
            return Collections.unmodifiableSet(unprocessedTimeouts);
        }
    }
    //定时任务的内部包装类，双向链表结构。会保存定时任务到期执行的任务、deadline、round等信息。
    private static final class HashedWheelTimeout implements Timeout {

        private static final int ST_INIT = 0;
        private static final int ST_CANCELLED = 1;
        private static final int ST_EXPIRED = 2;
        // 用来CAS方式更新定时任务状态
        private static final AtomicIntegerFieldUpdater<HashedWheelTimeout> STATE_UPDATER =
                AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimeout.class, "state");

        private final HashedWheelTimer timer;
        //具体到期要执行的任务
        private final TimerTask task;
        private final long deadline;

        @SuppressWarnings({"unused", "FieldMayBeFinal", "RedundantFieldInitialization"})
        private volatile int state = ST_INIT;

        /**
         * RemainingRounds will be calculated and set by Worker.transferTimeoutsToBuckets() before the
         * HashedWheelTimeout will be added to the correct HashedWheelBucket.
         */
        //离任务执行的轮数 任务加入后是计算这个值，每过一轮，该值-1
        long remainingRounds;

        /**
         * This will be used to chain timeouts in HashedWheelTimerBucket via a double-linked-list.
         * As only the workerThread will act on it there is no need for synchronization / volatile.
         */
        // 双向链表结构，由于只有worker线程会访问，这里不需要synchronization / volatile
        HashedWheelTimeout next;
        HashedWheelTimeout prev;

        /**
         * The bucket to which the timeout was added
         */
        //定时任务所在的槽
        HashedWheelBucket bucket;

        HashedWheelTimeout(HashedWheelTimer timer, TimerTask task, long deadline) {
            this.timer = timer;
            this.task = task;
            this.deadline = deadline;
        }

        @Override
        public Timer timer() {
            return timer;
        }

        @Override
        public TimerTask task() {
            return task;
        }

        @Override
        public boolean cancel() {
            // only update the state it will be removed from HashedWheelBucket on next tick.
            //这里只是修改状态为取消，实际会在下次tick的时候移除
            if (!compareAndSetState(ST_INIT, ST_CANCELLED)) {
                return false;
            }
            // If a task should be canceled we put this to another queue which will be processed on each tick.
            // So this means that we will have a GC latency of max. 1 tick duration which is good enough. This way
            // we can make again use of our MpscLinkedQueue and so minimize the locking / overhead as much as possible.
            // 加入到时间轮的待取消队列，并在每次tick的时候，从相应格子中移除。
            timer.cancelledTimeouts.add(this);
            return true;
        }

        void remove() {
            HashedWheelBucket bucket = this.bucket;
            if (bucket != null) {
                bucket.remove(this);
            } else {
                timer.pendingTimeouts.decrementAndGet();
            }
        }

        public boolean compareAndSetState(int expected, int state) {
            return STATE_UPDATER.compareAndSet(this, expected, state);
        }

        public int state() {
            return state;
        }

        @Override
        public boolean isCancelled() {
            return state() == ST_CANCELLED;
        }

        @Override
        public boolean isExpired() {
            return state() == ST_EXPIRED;
        }
        //过期并执行任务
        public void expire() {
            if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
                return;
            }

            try {
                task.run(this);
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
                }
            }
        }

        @Override
        public String toString() {
            final long currentTime = System.nanoTime();
            long remaining = deadline - currentTime + timer.startTime;
            String simpleClassName = ClassHelper.simpleClassName(this.getClass());

            StringBuilder buf = new StringBuilder(192)
                    .append(simpleClassName)
                    .append('(')
                    .append("deadline: ");
            if (remaining > 0) {
                buf.append(remaining)
                        .append(" ns later");
            } else if (remaining < 0) {
                buf.append(-remaining)
                        .append(" ns ago");
            } else {
                buf.append("now");
            }

            if (isCancelled()) {
                buf.append(", cancelled");
            }

            return buf.append(", task: ")
                    .append(task())
                    .append(')')
                    .toString();
        }
    }

    /**
     * Bucket that stores HashedWheelTimeouts. These are stored in a linked-list like datastructure to allow easy
     * removal of HashedWheelTimeouts in the middle. Also the HashedWheelTimeout act as nodes themself and so no
     * extra object creation is needed.
     */
    //HashedWheelBucket用来存放HashedWheelTimeout，结构类似于LinkedList。
    // 提供了expireTimeouts(long deadline)方法来过期并执行格子中的定时任务
    private static final class HashedWheelBucket {

        /**
         * Used for the linked-list datastructure
         */
        private HashedWheelTimeout head;
        private HashedWheelTimeout tail;

        /**
         * Add {@link HashedWheelTimeout} to this bucket.
         */
        void addTimeout(HashedWheelTimeout timeout) {
            assert timeout.bucket == null;
            timeout.bucket = this;
            if (head == null) {
                head = tail = timeout;
            } else {
                tail.next = timeout;
                timeout.prev = tail;
                tail = timeout;
            }
        }

        /**
         * Expire all {@link HashedWheelTimeout}s for the given {@code deadline}.
         */
        // 过期并执行格子中的到期任务，tick到该格子的时候，worker线程会调用这个方法，
        //根据deadline和remainingRounds判断任务是否过期
        void expireTimeouts(long deadline) {
            HashedWheelTimeout timeout = head;

            // process all timeouts
            //遍历格子中的所有定时任务
            while (timeout != null) {
                // 先保存next，因为移除后next将被设置为null
                HashedWheelTimeout next = timeout.next;
                if (timeout.remainingRounds <= 0) {
                    next = remove(timeout);
                    if (timeout.deadline <= deadline) {
                        timeout.expire();
                    } else {
                        // The timeout was placed into a wrong slot. This should never happen.
                        //不可能发生的情况，就是说round已经为0了，deadline却>当前槽的deadline
                        throw new IllegalStateException(String.format(
                                "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                    }
                } else if (timeout.isCancelled()) {
                    next = remove(timeout);
                } else {
                    //没有到期。论数-1
                    timeout.remainingRounds--;
                }
                timeout = next;
            }
        }

        public HashedWheelTimeout remove(HashedWheelTimeout timeout) {
            HashedWheelTimeout next = timeout.next;
            // remove timeout that was either processed or cancelled by updating the linked-list
            if (timeout.prev != null) {
                timeout.prev.next = next;
            }
            if (timeout.next != null) {
                timeout.next.prev = timeout.prev;
            }

            if (timeout == head) {
                // if timeout is also the tail we need to adjust the entry too
                if (timeout == tail) {
                    tail = null;
                    head = null;
                } else {
                    head = next;
                }
            } else if (timeout == tail) {
                // if the timeout is the tail modify the tail to be the prev node.
                tail = timeout.prev;
            }
            // null out prev, next and bucket to allow for GC.
            timeout.prev = null;
            timeout.next = null;
            timeout.bucket = null;
            timeout.timer.pendingTimeouts.decrementAndGet();
            return next;
        }

        /**
         * Clear this bucket and return all not expired / cancelled {@link Timeout}s.
         */
        void clearTimeouts(Set<Timeout> set) {
            for (; ; ) {
                HashedWheelTimeout timeout = pollTimeout();
                if (timeout == null) {
                    return;
                }
                if (timeout.isExpired() || timeout.isCancelled()) {
                    continue;
                }
                set.add(timeout);
            }
        }

        //轮询超时
        private HashedWheelTimeout pollTimeout() {
            HashedWheelTimeout head = this.head;
            if (head == null) {
                return null;
            }
            HashedWheelTimeout next = head.next;
            if (next == null) {
                tail = this.head = null;
            } else {
                this.head = next;
                next.prev = null;
            }

            // null out prev and next to allow for GC.
            head.next = null;
            head.prev = null;
            head.bucket = null;
            return head;
        }
    }

    private boolean isWindows() {
        return System.getProperty("os.name", "").toLowerCase(Locale.US).contains("win");
    }
}


```
