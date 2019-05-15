### Java 原子操作

#### 一、Unsafe了解

1、Unsafe类创建方法

>jdk API
```java
private static final Unsafe unsafe = Unsafe.getUnsafe();

@CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();
        if (var0.getClassLoader() != null) {
            throw new SecurityException("Unsafe");
        } else {
            return theUnsafe;
        }
    }
```

>利用反射
```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe=(Unsafe) f.get(null);
```

2、CAS操作
```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    private volatile int value;
    //类型变量的偏移量
    static {
          try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
          } catch (Exception ex) { throw new Error(ex); }
        }
        
    //compareAndSwapInt方法    
    unsafe.compareAndSwapInt(this, valueOffset, expect, update);
```

3、利用Unsafe如何使用堆外内存
>jdk 1.8 元空间

>Unsafe API 
```java
//申请内存 var1  表示向内存请求分配的空间大小
public native long allocateMemory(long var1);


public native void freeMemory(long var1);
```

4、Unsafe核心方法

> CAS 利弊

>park/unpark  wait()/notify()

>内存分配




#### 二、java.util.concurrent.atomic包中的类

1、各个类的作用及应用场景


2、1.8新特性


#### 三、ABA 相关问题

>什么是ABA问题

>ABA有哪些危害

>ABA解决方案

#### 四、伪共享问题
![缓存架构](https://assets.2dfire.com/frontend/5452a490625121da8e0b3c15a2992e00.png)
>CPU缓存架构

>CPU缓存行

>伪共享产生的原因

>伪共享如何解决和避免

#### 五、内存屏障
> 内存和CPU

>内存屏障是什么



关于原子类的问题：
>1、Unsafe是什么？

>2、Unsafe为什么是不安全的？

>3、Unsafe的实例怎么获取？

>4、Unsafe的CAS操作？

>5、Unsafe的阻塞/唤醒操作？

>6、Unsafe实例化一个类？

>7、实例化类的六种方式？

>8、原子操作是什么？

>9、原子操作与数据库ACID中A的关系？

>10、AtomicInteger怎么实现原子操作的？

>11、AtomicInteger主要解决了什么问题？

>12、AtomicInteger有哪些缺点？

>13、ABA是什么？

>14、ABA的危害？

>15、ABA的解决方法？

>16、AtomicStampedReference是怎么解决ABA的？

>17、实际工作中遇到过ABA问题吗？

>18、CPU的缓存架构是怎样的？

>19、CPU的缓存行是什么？

>20、内存屏障又是什么？

>21、伪共享是什么原因导致的？

>22、怎么避免伪共享？

>23、消除伪共享在java中的应用？

>24、LongAdder的实现方式？

>25、LongAdder是怎么消除伪共享的？

>26、LongAdder与AtomicLong的性能对比？

>27、LongAdder中的cells数组是无限扩容的吗




参考链接：

[Unsafe解析](https://mp.weixin.qq.com/s/0s-u-MysppIaIHVrshp9fA)

[java并发包之AtomicInteger源码分析](https://mp.weixin.qq.com/s/DdwSC5bYgFCWwnb0jxkspg)

[java并发包之AtomicStampedReference源码分析](https://mp.weixin.qq.com/s/7pY1jKNVB_dvadZRIzmD1Q)

[什么是伪共享（false sharing）](https://mp.weixin.qq.com/s/rd13SOSxhLA6TT13N9ni8Q)

[java并发包之LongAdder源码分析](https://mp.weixin.qq.com/s/_-z1Bz2iMiK1tQnaDD4N6Q)

[内存屏障](https://www.jianshu.com/p/2ab5e3d7e510)