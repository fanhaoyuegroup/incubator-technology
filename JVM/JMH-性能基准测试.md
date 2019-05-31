JMH-性能基准测试

## 简介
JMH是一种Java工具，用于构建，运行和分析用Java和其他语言编写的针对JVM的nano/micro/milli/macro基准测试。。简单地说就是在 method 层面上的 benchmark，精度可以精确到微秒级。可以看出JMH主要使用在当你已经找出了热点函数，而需要对热点函数进行进一步的优化时，就可以使用 JMH 对优化的效果进行定量的分析。
## 使用场景

1：客观知道某个方法某个函数执行时间，以及执行时间与参数的相关性。

2：客观比较同一种方案两种不同实现方式的优略。当前前提是你足够了解他们的场景（比如A算法在数据量大的情况下展现出较好的性能，而B算法在数据量小的时候展现出较好的性能），你的测试用例足够正确的情况下才能得出较为正确的结论。
## start
####   引入JMH依赖
```

        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>${jmh.version}</version>
        </dependency>
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>${jmh.version}</version>
            <scope>provided</scope>
        </dependency>

```

#### 第一个例子🌰
```java
/**
 * Description:
 * Created by CalvinKris
 * Since 2019/5/29 23:50 PM
 */
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@State(Scope.Benchmark)
public class NormalizeTicksPerWheelTest {


    @Param({"7", "9", "17", "19", "29", "39"})
    int testNum;


    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public int oldNormalizeTicksPerWheel() {
        int normalizedTicksPerWheel = 1;
        while (normalizedTicksPerWheel < testNum) {
            normalizedTicksPerWheel <<= 1;
        }
        return normalizedTicksPerWheel;
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public int newNormalizeTicksPerWheel() {
        int n = testNum - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return n + 1;
    }
    //启动选项
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(NormalizeTicksPerWheel.class.getSimpleName()) //benchmark 所在的类的名字，注意这里是使用正则表达式对所有类进行匹配的。                                                       
                .forks(1) //进行 fork 的次数。如果 fork 数是2的话，则 JMH 会 fork 出两个进程来进行测试
                .build();

        new Runner(opt).run();
    }
}

```
运行结果,显示各项指标
```

Result "com.kris.microbench.DemoTest.oldNormalizeTicksPerWheel":
  0.004 ±(99.9%) 0.001 us/op [Average]
  (min, avg, max) = (0.004, 0.004, 0.004), stdev = 0.001
  CI (99.9%): [0.004, 0.004] (assumes normal distribution)


# JMH version: 1.21
# VM version: JDK 1.8.0_191, Java HotSpot(TM) 64-Bit Server VM, 25.191-b12
# VM invoker: /Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home/jre/bin/java
# VM options: -javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=55085:/Applications/IntelliJ IDEA.app/Contents/bin -Dfile.encoding=UTF-8
# Warmup: 5 iterations, 1 s each
# Measurement: 5 iterations, 1 s each
# Timeout: 10 min per iteration
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Average time, time/op
# Benchmark: com.kris.microbench.DemoTest.oldNormalizeTicksPerWheel
# Parameters: (testNum = 39)

# Run progress: 91.67% complete, ETA 00:00:10
# Fork: 1 of 1
# Warmup Iteration   1: 0.004 us/op
# Warmup Iteration   2: 0.005 us/op
# Warmup Iteration   3: 0.004 us/op
# Warmup Iteration   4: 0.004 us/op
# Warmup Iteration   5: 0.004 us/op
Iteration   1: 0.004 us/op
Iteration   2: 0.004 us/op
Iteration   3: 0.004 us/op
Iteration   4: 0.004 us/op
Iteration   5: 0.004 us/op


Result "com.kris.microbench.DemoTest.oldNormalizeTicksPerWheel":
  0.004 ±(99.9%) 0.001 us/op [Average]
  (min, avg, max) = (0.004, 0.004, 0.004), stdev = 0.001
  CI (99.9%): [0.004, 0.004] (assumes normal distribution)


# Run complete. Total time: 00:02:09

REMEMBER: The numbers below are just data. To gain reusable insights, you need to follow up on
why the numbers are the way they are. Use profilers (see -prof, -lprof), design factorial
experiments, perform baseline and negative tests that provide experimental control, make sure
the benchmarking environment is safe on JVM/OS/HW level, ask for reviews from the domain experts.
Do not assume the numbers tell you what you want them to tell.

Benchmark                           (testNum)  Mode  Cnt  Score    Error  Units
DemoTest.newNormalizeTicksPerWheel          7  avgt    5  0.003 ±  0.001  us/op
DemoTest.newNormalizeTicksPerWheel          9  avgt    5  0.003 ±  0.001  us/op
DemoTest.newNormalizeTicksPerWheel         17  avgt    5  0.003 ±  0.001  us/op
DemoTest.newNormalizeTicksPerWheel         19  avgt    5  0.003 ±  0.001  us/op
DemoTest.newNormalizeTicksPerWheel         29  avgt    5  0.003 ±  0.001  us/op
DemoTest.newNormalizeTicksPerWheel         39  avgt    5  0.003 ±  0.001  us/op
DemoTest.oldNormalizeTicksPerWheel          7  avgt    5  0.003 ±  0.001  us/op
DemoTest.oldNormalizeTicksPerWheel          9  avgt    5  0.004 ±  0.001  us/op
DemoTest.oldNormalizeTicksPerWheel         17  avgt    5  0.004 ±  0.002  us/op
DemoTest.oldNormalizeTicksPerWheel         19  avgt    5  0.004 ±  0.001  us/op
DemoTest.oldNormalizeTicksPerWheel         29  avgt    5  0.004 ±  0.001  us/op
DemoTest.oldNormalizeTicksPerWheel         39  avgt    5  0.004 ±  0.001  us/op

```
## 用法
Mode
Mode 表示 JMH 进行 Benchmark 时所使用的模式。通常是测量的维度不同，或是测量的方式不同。目前 JMH 共有四种模式：

Throughput: 整体吞吐量，例如“1秒内可以执行多少次调用”。
AverageTime: 调用的平均时间，例如“每次调用平均耗时xxx毫秒”。
SampleTime: 随机取样，最后输出取样结果的分布，例如“99%的调用在xxx毫秒以内，99.99%的调用在xxx毫秒以内”
SingleShotTime: 以上模式都是默认一次 iteration 是 1s，唯有 SingleShotTime 是只运行一次。往往同时把 warmup 次数设为0，用于测试冷启动时的性能。

@OutputTimeUnit
benchmark 结果所使用的时间单位。
Iteration
Iteration 是 JMH 进行测试的最小单位。在大部分模式下，一次 iteration 代表的是一秒，JMH 会在这一秒内不断调用需要 benchmark 的方法，然后根据模式对其采样，计算吞吐量，计算平均执行时间等。

Warmup
Warmup 是指在实际进行 benchmark 前先进行预热的行为。为什么需要预热？因为 JVM 的 JIT 机制的存在，如果某个函数被调用多次之后，JVM 会尝试将其编译成为机器码从而提高执行速度。所以为了让 benchmark 的结果更加接近真实情况就需要进行预热。
@Measurement
实际迭代测量的次数
注解
现在来解释一下上面例子中使用到的注解，其实很多注解的意义完全可以望文生义 :)

@Benchmark
表示该方法是需要进行 benchmark 的对象，用法和 JUnit 的 @Test 类似。

@Mode
Mode 如之前所说，表示 JMH 进行 Benchmark 时所使用的模式。

@State
State 用于声明某个类是一个“状态”，然后接受一个 Scope 参数用来表示该状态的共享范围。因为很多 benchmark 会需要一些表示状态的类，JMH 允许你把这些类以依赖注入的方式注入到 benchmark 函数里。Scope 主要分为两种。

Thread: 该状态为每个线程独享。
Benchmark: 该状态在所有线程间共享。
@Param
@Param 可以用来指定某项参数的多种情况。特别适合用来测试一个函数在不同的参数输入的情况下的性能。

@Setup
@Setup 会在执行 benchmark 之前被执行，正如其名，主要用于初始化。

@TearDown
@TearDown 和 @Setup 相对的，会在所有 benchmark 执行结束以后执行，主要用于资源的回收等。



CompilerControl
控制 compiler 的行为，例如强制 inline，不允许编译等。

Group
可以把多个 benchmark 定义为同一个 group，则它们会被同时执行，主要用于测试多个相互之间存在影响的方法。

Level
用于控制 @Setup，@TearDown 的调用时机，默认是 Level.Trial，即benchmark开始前和结束后。

Profiler
JMH 支持一些 profiler，可以显示等待时间和运行时间比，热点函数等。

[参考资料JMH - Java Microbenchmark Harness](http://tutorials.jenkov.com/java-performance/jmh.html)


