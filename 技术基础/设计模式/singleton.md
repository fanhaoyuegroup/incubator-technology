## 单例模式
* 概念：程序某个对象只可能存在一个，那我们称这个类为单例类
* 作用：减少内存紧张问题，并且管理简单
* 分类：饿汉模式和懒汉模式
* 区别：
    * 前者在类初始化就在堆内存就创建，后者在调用getInstance()方法时初始化
    * 详细讲解：
        * 由于前者在声明时用static修饰，所以我们可以直接通过ClassName.singleton拿到对象的引用
        * 而后者不用声明static修饰，同时默认赋值为NULL，所以必须通过getInstance()方法拿到对象
        * 二者都可以通过反射破坏单例模式
        * 饿汉模式为线程安全，懒汉模式线程不安全
* 单例模式-饿汉模式demo
   ```java
   public class Singleton{
      private static Singleton singleton = new Singleton();
      private Singleton(){}
      public static Singleton getInstance(){
        return singleton;
      }  
    }
    ```
* 单例模式-懒汉模式（错误写法）
    ```java
    public class ErrorSingleton{
      private ErrorSingleton errorSingleton = null;
      private ErrorSingleton(){}
      public ErrorSingleton getErrorSingleton(){
          if(errorSingleton == null){
              errorSingleton = new ErrorSingleton();  
          }
          return errorSingleton;
      }
    }
    ```
* 通过反射破坏单例规则
    1. 通过Class拿到构造器对象
    2. 设置setAccessible(true)
    3. 通过构造器对象生成一个实例
* 多线程单例
    1. 使用synchronized关键字锁住getInstacne()方法，这种方案锁粒度太粗，影响性能
    2. 使用synchronized锁住new方法，双重校验，并且引用要有volatile修饰，防止内存重排序
    3. demo
    ```java
    public class Singleton{
      private static volatile Singleton singleton;
      private Singleton(){}
      public static Singleton getInstance(){
          if(singleton == null){
              synchronized (Singleton.class){
                  if(singleton == null){
                      singleton = new Singleton(); 
                  }  
              } 
          }  
        return singleton;
      }
    }
    ```
* 总结
   -
   * 设计单例最佳为ENUM类
