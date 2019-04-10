
### 一、内存分配策略

我们先来看一个题目，你知道正确的运行结果并给出解释吗。不知道也没关系，我会在下面给出具体的分析。

```java
    @Test
    public void test() {
        String s1 = "abc";
        String s2 = "abc";

        String s3 = new String("abc");
        String s4 = new String("abc");

        System.out.println(s1 == s2);     // true
        System.out.println(s3 == s4);     // false
        System.out.println(s1 == s3);     // false
    }
```
`String s = ""` 与 `String s = new String("")` 两种方式都可以创建字符串对象，它们有什么不同吗。

**1.1 创建对象的方式**

`String s = "abc"` 方式创建的对象，存储在字符串常量池中，在创建字符串对象之前，会先在常量池中检查是否存在 `abc` 对象。如果存在，则直接返回常量池中 `abc`对象的引用，不存在会创建该对象，并将该对象的引用返回给对象 `s`。

以 HotSpot 虚拟机为例，在 jdk1.8 之前，字符串常量池在方法区中，为了减小方法区内存溢出的风险，在 jdk1.8 之后就把字符串常量池转移到 java 堆中了。

`String s = new String("abc")` 这种方式，实际上 `abc` 本身就是字符串池中的一个对象，在运行 `new String()` 时，把字符串常量池中的字符串  `abc` 复制到堆中，因此该方式不仅会在堆中，还会在常量池中创建 `abc` 字符串对象。 最后把 java 堆中对象的引用返回给  `s`。

**1.2 问题分析**

通过上面的分析，大家应该对问题结果有一个感性的认识了。

`s1`  指向字符串常量池中的 `abc` 对象，`s2`  也指向字符串常量池中的 `abc` 对象，因此 `s1` 与 `s2` 指向的是同一个对象，故 `s1 == s2` 返回 `true`。

`s3` 与 `s4` 分别指向 java 堆中两个不同的对象，因此 `s3 == s4` 是 `false`。

`s1` 与 `s3` 分别指向字符串常量池中的对象与 java 堆中的对象，`s1 == s3` 返回值也是 `false`。

### 二、问题扩展
上面的问题清楚了原理还很好理解，下面还有一个例子，理解起来就不那么容易了。

```java
    @Test
    public void test() {
        String s1 = "abc";
        String s2 = "a";
        
        System.out.println(s1 == ("a" + "bc"));    // true
        System.out.println(s1 == (s2 + "bc"));     // false 
    }
```

**2.1 字符串常量重载 "+"**

`"a" + "bc"` 是两个字符串常量的拼接，当一个字符串由多个字符串常量连接而成时，它自己也肯定是字符串常量。java 虚拟机对于字符串常量的 "+" 号连接，在程序编译期，java 虚拟机就将常量字符串的 "+" 连接优化为连接后的值。 

这样一来，最终只会在常量池中创建一个 `abc` 对象，并没有创建临时字符串对象 `a` 和 `bc`，减轻了垃圾收集器的压力。`s1 == ("a" + "bc")`，因为 `abc`（`s1`） 在字符串常量池中已经存在了，因此返回值是 `true`。

**2.2 字符串引用重载 "+"**

java 虚拟机对于有字符串引用存在的字符串连接，即 `s2 + "bc"` 在被编译器执行的时候，会自动引入 `StringBuilder` 对象，调用其 `append()` 方法，最终调用 `toString()` 方法返回其在堆中对象的引用。 下面是反编译后的部分字节码。
<center>

![这里写图片描述 ](https://img-blog.csdn.net/201808231808049)
</center>

因此 `s2 + "bc"` 就等同于下面过程。

```java
s2 + "bc" = new StringBuilder().append(s2).append("bc").toString();
```
`s1 == (s2 + "bc")` 在进行比较时，`s1` 是字符串常量池中对象的引用，`(s2 + "bc")` 则是 java 堆中一个对象的引用，因此返回值是 `false`。为什么说 `s2 + "bc"` 返回的是堆中对象的引用呢，只要到 `StringBuilder` 源码中查看一下 `toString()` 就能理解了。

```java
    /**
    * StringBuilder 类的 toSTring() 方法
    */
    @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
```

### 三、参考资料
 - [String 对象内存分配 (常量池和堆的分配)](https://blog.csdn.net/Mypromise_TFS/article/details/81504137)
 - [Java—String 字符串运算符"+"重载分析 ](https://blog.csdn.net/whp1473/article/details/79920950)