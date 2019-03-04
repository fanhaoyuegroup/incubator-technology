### 一、策略模式概述

**1.1 什么是策略模式**

策略模式 (Strategy Pattern) 是指对一系列的算法定义，并将每一个算法封装起来，而且使它们还可以相互替换。此模式让算法的变化独立于使用算法的客户。

**1.2 策略模式组成结构**

 - <font size=3>环境 (Context)：持有一个策略类的引用，最终给客户端调用。
 - <font size=3>抽象策略 (Strategy)： 策略类，通常是一个接口或者抽象类。
 - <font size=3>具体策略 (ConcreteStrategy)：实现了策略类中的策略方法，封装相关的算法和行为。

**1.3 策略模式 UML 图解**
<img src="http://img.blog.csdn.net/20180116173416405?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZWphcw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="100%">

**1.4 策略模式应用场景**

 - 多个类只区别在表现行为不同，可以使用 Strategy 模式，在运行时动态选择具体要执行的行为。
 - 需要在不同情况下使用不同的策略 (算法)，或者策略还可能在未来用其它方式来实现。
 - 对客户隐藏具体策略 (算法) 的实现细节，彼此完全独立。

### 二、策略模式案例

**2.1 问题描述** 

模拟鸭子游戏：游戏中会出现各种鸭子，一边游泳戏水、一边呱呱叫，为了提高游戏的乐趣，加入了让鸭子飞的功能。但是考虑到并不是所有的鸭子都会飞，比如下面这种橡皮鸭。现在让你利用 OO 技术，设计鸭子相关的类。

<img src="http://img.blog.csdn.net/20180116195031135?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZWphcw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" width="10%">

**2.2 策略模式代码实现**

抽象策略接口 `FlyBehavior`

``` java
package com.jas.strategy;

public interface FlyBehavior {
    void fly();
}
```

具体策略类 `FlyWithWings`(实现鸭子飞)

``` java
package com.jas.strategy;

public class FlyWithWings implements FlyBehavior {
    @Override
    public void fly() {
        System.out.println("I'm flying!");
    }
}
```

具体策略类 `FlyNoWay`(实现鸭子不会飞)

``` java
package com.jas.strategy;

public class FlyNoWay implements FlyBehavior {
    @Override
    public void fly() {
        System.out.println("I can't fly!");
    }
}
```

抽象环境 `Duck` 类

``` java
package com.jas.strategy;

public abstract class Duck {
    private FlyBehavior flyBehavior;
    
    public void swim(){
        System.out.println("All ducks float.");
    }
    
    public abstract void display();

    public void performFly(){
        flyBehavior.fly();
    }
    
    public void setFlyBehavior(FlyBehavior flyBehavior){
        this.flyBehavior = flyBehavior;
    }
    
}
```

具体环境测试类 `RubberDuck`

``` java
package com.jas.strategy;

public class RubberDuck extends Duck {

    @Override
    public void display() {
        System.out.println("Rubber Duck");
    }
    
    public static void main(String[] args) {
        Duck rubberDuck = new RubberDuck();    //橡皮鸭实例
        rubberDuck.setFlyBehavior(new FlyNoWay());    //橡皮鸭不会飞
        rubberDuck.performFly();
    }
}

// 输出
// I'm rubber duck.
// I can't fly!
```

具体环境测试类 `TrueDuck`

``` java
package com.jas.strategy;

public class TrueDuck extends Duck{
    @Override
    public void display() {
        System.out.println("I'm true duck.");
    }

    public static void main(String[] args) {
        Duck trueDuck = new TrueDuck();
        trueDuck.display();
        trueDuck.setFlyBehavior(new FlyWithWings());
        trueDuck.performFly();
    }
}

//输出
// I'm true duck.
// I'm flying!
```

**2.3 类图总结**

![](https://image-static.segmentfault.com/622/685/622685367-5c7ccbf692d5c_articlex)

### 三、总结

**3.1 策略模式的优缺点**

优点

 - 策略模式提供了管理相关的算法族的办法，从而避免重复的代码。
 - 策略模式提供了可以替换继承关系的办法。因为继承使得动态改变算法或行为变得不可能。
 - 使用策略模式可以避免使用多重条件转移语句。

缺点

 - 客户端必须知道所有的策略类，并自行决定使用哪一个策略类。
 - 策略模式造成很多的策略类，每个具体策略类都会产生一个新类。

**3.2 参考资料**

《Head First 设计模式》