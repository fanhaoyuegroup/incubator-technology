## 建造者模式

### 一、`概念`
1、又名生成器模式，是一种对象构建模式。它可以将复杂对象的建造过程抽象出来（抽象类别），使这个抽象过程的不同实现方法可以构造出不同表现（属性）的对象。

### 二、`为什么使用该模式（优点）`
1、客户端不必知道产品内部组成的细节，将产品本身与产品的创建过程解耦，使得相同的创建过程可以创建不同的产品对象。
2、用户使用不同的具体建造者即可得到不同的产品对象。

### 三、`对比抽象工厂模式`
1、抽象工厂模式实现对产品家族的创建，一个产品家族是这样的一系列产品：具有不同分类维度的产品组合，采用抽象工厂模式不需要关心构建过程，只关心什么产品由什么工厂生产即可。而建造者模式则是要求按照指定的蓝图建造产品，它的主要目的是通过`组装零配件`而产生一个新产品。

### 四、`UML类图`
![image](https://user-gold-cdn.xitu.io/2018/6/3/163c4f8d2ab77ee7?w=976&h=454&f=png&s=114928)

### 五、`四个角色`
1. Product（产品角色）： 一个具体的产品对象。
2. Builder（抽象建造者）： 创建一个Product对象的各个部件指定的抽象接口。
3. ConcreteBuilder（具体建造者）： 实现抽象接口，构建和装配各个部件。
4. Director（指挥者）： 构建一个使用Builder接口的对象。它主要是用于创建一个复杂的对象。它主要有两个作用，一是：隔离了客户与对象的生产过程，二是：负责控制产品对象的生产过程。

### 六、`demo示例`

[建造者模式demo](https://github.com/fanhaoyuegroup/interest-group/tree/master/design-pattern/src/main/java/com/fan/design/build)

### 七、`主流框架使用`
1. Guvva缓存
2. Spring
3. [Mybatis使用建造者模式](https://cloud.tencent.com/developer/article/1330373)