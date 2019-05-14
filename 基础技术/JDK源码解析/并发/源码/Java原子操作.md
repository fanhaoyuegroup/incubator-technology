### 一、Java 原子操作

#### 二、Unsafe了解

1、Unsafe类创建方法

2、CAS操作

3、利用Unsafe如何使用堆外内存

4、Unsafe核心方法


#### 三、java.util.concurrent.atomic包中的类

1、各个类的作用及应用场景

2、1.8新特性


#### 四、ABA 相关问题

#### 五、伪共享问题


#### 六、内存屏障



关于原子类的问题，笔者整理了大概有以下这些：
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