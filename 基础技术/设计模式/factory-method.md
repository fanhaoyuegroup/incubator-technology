### 一、什么是工厂方法

**1.1 工厂方法模式**

工厂方法模式定义了一个创建对象的接口，但由子类决定要实例化的类是哪一个。工厂方法让类把实例化推迟到子类。

**1.2 工厂方法模式组成结构**

 - 抽象工厂角色：是具体工厂角色必须实现的接口或者必须继承的父类，在 java 中它由抽象类或者接口来实现，声明了抽象的工厂方法
 - 具体工厂角色：它含有和具体业务逻辑有关的代码，由应用程序调用以创建对应具体产品的对象，实现了父类或接口中定义的抽象工厂方法 
 - 抽象产品角色：它是具体产品继承的父类或者是实现的接口，在 java 中一般由抽象类或者接口来实现 
 - 具体产品角色：具体工厂角色所创建的对象就是此角色的实例，在 java 中由具体的类来实现 

**1.3 工厂方法模式的 UML 图解** 

![](https://img-blog.csdn.net/20180121150356757)

### 二、工厂方法实例

**2.1 抽象工厂例子**

下面是一个用工厂方法实现的日志记录器。

抽象日志接口 `Logger`

``` java
public interface Logger {

    /**
     * 写日志
     */
    void writeLog();
}
```

具体日志接口实现类 `DatabaseLogger`

```java
public class DatabaseLogger implements Logger {
    @Override
    public void writeLog() {
        System.out.println("database log...");
    }
}
```

具体日志接口实现类 `FileLogger`

``` java
public class FileLoggerFactory implements LoggerFactory {
    @Override
    public Logger creatLogger() {
        return new FileLogger();
    }
}
```

抽象日志工厂接口 `LoggerFactory`

``` java
public interface LoggerFactory {

    /**
     * 创建日志对象的工厂方法
     *
     * @return 日志对象
     */
    Logger creatLogger();
}
```

具体日志工厂 `DatabaseLoggerFactory` 类

``` java
public class DatabaseLoggerFactory implements LoggerFactory {
    @Override
    public Logger creatLogger() {
        return new DatabaseLogger();
    }
}
```

具体日志工厂 `DatabaseLoggerFactory` 类

``` java
public class FileLoggerFactory implements LoggerFactory {
    @Override
    public Logger creatLogger() {
        return new FileLogger();
    }
}
```

测试类

```java
public class LoggerDriven {
    public static void main(String[] args) {
        Logger logger;
        LoggerFactory loggerFactory;

        /*loggerFactory = new DatabaseLoggerFactory();
        logger = loggerFactory.creatLogger();*/

        loggerFactory = new FileLoggerFactory();
        logger = loggerFactory.creatLogger();
        logger.writeLog();
    }
}

// 输出：file log...
```

**2.2 真实世界的工厂方法**

`AbstractCollection` 类中有一个 `iterator` 的抽象方法，有很多子类在继承该类的时候都实现了 `iterator` 方法，该方法的目的就是用于返回生产的迭代器对象。

``` java
public abstract Iterator<E> iterator();
```

`ArrayDeque` 中的 `iterator` 方法

``` java
    public Iterator<E> iterator() {
        return new DeqIterator();
    }
```

`TreeMap` 中 `Value` 的 `iterator` 方法

``` java
    class Values extends AbstractCollection<V> {
        public Iterator<V> iterator() {
            return new ValueIterator(getFirstEntry());
        }
        ....
    }
```

### 三、工厂方法总结

**3.1 工厂方法优点**

 - 向客户隐藏了某种具体产品类的实现细节，用户只需要关心对应的工厂即可
 - 在系统中加入新产品时，无需修改顶层抽象工厂（无需修改客户端），只需要添加一个具体工厂就可以了，增强了系统的扩展性
 - 基于多态特性设计，可以抽象的创建某个产品对象
 
**3.2 工厂方法缺点**

每有一个具体产品时就需要创建一个工厂，在产品过多的情况下会产生类爆炸的情况，增加系统的复杂性。

**3.3 适用场景**

 - 客户端不需要知道具体产品类的类名(多态)，只需要知道所对应的工厂
 - 一个类通过其子类来指定创建哪个对象
 
### 参考资料

《设计模式的艺术》