#  桥接模式

## 定义与类型

### 定义

将抽象部分与它的具体实现部分分离开来，使它们都可以独立地变化。桥接模式将继承关系转化成关联关系，它降低了类与类之间的耦合度，减少了系统中类的数量，防止类爆炸。

```
将抽象部分与它的具体实现部分分离，其实这并不是将抽象类与他的派生类分离，而是说在一个系统的抽象化和实现化之间使用关联关系（组合或者聚合关系）而不是继承关系，从而使两者可以相对独立地变化。
```

### 使用场景

1. 抽象和具体实现之间增加更多的灵活性

   ```
   使用桥接模式就可以避免在这两个层次之间建立静态的继承关系，而是建立关联关系。此外，抽象部分和具体实现部分，它们都可以分别通过继承关系独立扩展，并且互不影响，就可以动态地将一个抽象化子类的对象和一个具体实现化子类的对象进行组合，这样就把抽象化角色和具体实现化角色实现了解耦。
   ```

2. 一个类存在两个（或多个）独立变化的维度，且这两个（或多个）维度都需要独立进行扩展

   ```
   抽象的部分可以独立扩展，具体实现也可以独立扩展
   ```

3. 不希望使用继承，或因为多层继承导致系统类的个数剧增

### 优点

1. 分离抽象部分及其具体实现部分

   ```
   因为桥接模式使用了组合，使用对象间的关联关系，来解耦了抽象和具体实现之间的固有绑定关系，使抽象和实现可以沿着各自的维度进行扩展、变化。也就是说，抽象和实现不在同一个继承层次结构中，从而通过组合来获得多维度的组合对象。
   ```

2. 提高了系统的可扩展性

   ```
   在两个变化维度中，扩展任意一个维度都不需要修改原有的系统
   ```

3. 符合开闭原则

4. 符合合成复用原则

### 缺点

1. 增加了系统的理解与设计难度

   ```
   由于类之间的关系建立在抽象层，要求我们在编码的时候，一开始就要针对抽象层进行设计和编程
   ```

2. 需要正确地识别出系统中两个独立变化的维度



 ### 相关设计模式

1. 组合模式

   ```
   组合模式更强调的是部分和整体间的组合，而桥接模式强调的是平行级别下不同类的组合
   ```

2. 适配器模式

   ```
   适配器模式和桥接模式都是为了让两个东西配合工作，但它们两个的目的不一样，适配器模式是改变已有的接口，让它们之间可以相互配合，而桥接模式是分离抽象和具体实现。也就是说，适配器模式可以把功能上相似但是接口不同的类适配起来，而桥接模式是把类的抽象和类的具体实现分离开，然后在此基础上让这些层次结构结合起来。
   ```


## 案例分析

1. 场景

   ```
   有两个银行，分别是ABC和ICBC银行，同时有两个账号，分别是定期账号和活期账号。现在银行要进行开户，并且开户后要可以查看开户类型。
   ```

2. 编码

   - 账号接口 Account

     ```Java
     public interface Account {
         /**
          * 开户
          */
         Account openAccount();
         /**
          * 开户类型
          */
         void showAccountType();
     }
     ```

     ​

   - 账号的两个实现类 SavingAccount和 DepositAccount

     ```Java
     public class SavingAccount implements Account {
         @Override
         public Account openAccount() {
             System.out.println("SavingAccount--开活期账号");
             return new SavingAccount();
         }

         @Override
         public void showAccountType() {
             System.out.println("SavingAccount--这是一个活期账号");
         }
     }
     ---
     public class DepositAccount implements Account {

         @Override
         public Account openAccount() {
             System.out.println("DepositAccount--开定期账号");
             return new DepositAccount();
         }

         @Override
         public void showAccountType() {
             System.out.println("DepositAccouont--这是一个定期账号");
         }
     }
     ```

     ​

   - 银行抽象类 Bank

     ``` java
     public abstract class Bank {

         /**
          * 这里要写成一个抽象的，因为要把Account引入到Bank里面，通
          * 过这种组合的方式，把Account的行为交给Bank的子类来实现，即
          * Bank这个抽象类中的某个行为要委托给Account这个接口的实现，
          * 抽象和具体的实现分离指定的就是这种情况。
          */

         /**
          * 要交给子类，声明为protected
          */
         protected Account account;

         /**
          * 通过构造器把Account传过来，也可以通过setter注入的方式赋值
          */
         public Bank(Account account){
            this.account = account;
         }
         /**
          *  这个方法参照Account接口中的方法功能，因为Bank里面的具体方法要委托给Account里面
          *  的openAccount方法。注意这里不要求方法名称和Account中一致
          */
         abstract Account openAccount();

     }
     ```

     ​

   - 银行抽象类的子类 ABCBank和ICBCBank

     ``` java
     public class ABCBank extends Bank  {

         /**
          * 
          * @param account
          */
         public ABCBank(Account account) {
             super(account);
         }

         /**
          * 这里返回的就是父类中的Account
          * @return
          */
         @Override
         Account openAccount() {
             System.out.println("ABCBank--开户中国农业银行账号");
             // 关键一点
             account.openAccount();
             return account;
         }
     }
     ---
     public class ICBCBank extends Bank {

         /**
          *  
          * @param account
          */
         public ICBCBank(Account account) {
             super(account);
         }

         /**
          * 这里返回的就是父类中的Account
          * @return
          */
         @Override
         Account openAccount() {
             System.out.println("ICBC--开户中国工商银行账号");
             // 关键一点
             account.openAccount();
             return account;
         }
     }
     ```

   - 单元测试 TestDemo

     ``` java
     public class TestDemo {

         public static void main(String[] args) {
             // ICBCBank-DepositAccount
             Bank icbcBank = new ICBCBank(new DepositAccount());
             Account icbcAccount = icbcBank.openAccount();
             System.out.println("**************************");
             icbcAccount.showAccountType();
             System.out.println("--------------------------");
             // ICBCBank-SavingAccount
             Bank icbcBank2 = new ICBCBank(new SavingAccount());
             Account icbcAccount2 = icbcBank2.openAccount();
             System.out.println("**************************");
             icbcAccount2.showAccountType();
             System.out.println("--------------------------");

             // ABCBank-DepositAccount
             Bank abcBank2 = new ABCBank(new DepositAccount());
             Account abcAccount2 = abcBank2.openAccount();
             System.out.println("**************************");
             abcAccount2.showAccountType();
             System.out.println("--------------------------");
             // ABCBank-SavingAccount
             Bank abcBank = new ABCBank(new SavingAccount());
             Account abcAccount = abcBank.openAccount();
             System.out.println("**************************");
             abcAccount.showAccountType();
         }
     }
     ```

   - 总结

     ![](img/bridge_01.jpg)

   ## 桥接模式在JDK源码中的应用
   
    > Java封装了JDBC，java.sql.Driver接口，它的的实现类就是我们说的驱动，如MySQL的Driver，Oracle的Driver等,
    它们都实现了JDBC的Driver接口。
    
    
   ### JDBC如何获取数据库连接
    
   - DriverManager类
   
   ``` java
   /**
    *使用集合保存注册进来的Driver，DriverInfo只是对Driver进行了简单地封装
    */
   CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
   
   /**
    * registerDriver 方法，用来注册驱动
    */
    // Class.forName("com.mysql.jdbc.Driver")
       public static synchronized void registerDriver(java.sql.Driver driver,
                        DriverAction da)
                throws SQLException {
            
             /* Register the driver if it has not already been added to our list */
              if(driver != null) {
                registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
                } else {
                    // This is for compatibility with the original DriverManager
                    throw new NullPointerException();
                  }
                println("registerDriver: " + driver);
             } 
    /**
     * getConnection方法，它获取的是JDBC提供的相同的接口Connection，不同的数据库厂商可以自由地实现这个接口，
     * 在调用的过程中只需要使用接口去定义就可以了，然后使用多态机制。Connection中包含了操作数据库的不同方法。
     */         
      @CallerSensitive
                   public static Connection getConnection(String url,
                       String user, String password) throws SQLException {
                       java.util.Properties info = new java.util.Properties();
               
                       if (user != null) {
                           info.put("user", user);
                       }
                       if (password != null) {
                           info.put("password", password);
                       }
               
                       return (getConnection(url, info, Reflection.getCallerClass()));
                   }
              ---
                  private static Connection getConnection(
                      String url, java.util.Properties info, Class<?> caller) throws SQLException {
                      ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
                      synchronized(DriverManager.class) {
                          // synchronize loading of the correct classloader.
                          if (callerCL == null) {
                              callerCL = Thread.currentThread().getContextClassLoader();
                          }
                      }
                      if(url == null) {
                          throw new SQLException("The url cannot be null", "08001");
                      } 
                      SQLException reason = null;
                      for(DriverInfo aDriver : registeredDrivers) {
                          if(isDriverAllowed(aDriver.driver, callerCL)) {
                              try {
                                  println("    trying " + aDriver.driver.getClass().getName());
                                  Connection con = aDriver.driver.connect(url, info);
                                  if (con != null) {
                                      // Success!
                                      println("getConnection returning " + aDriver.driver.getClass().getName());
                                      return (con);
                                  }
                              } catch (SQLException ex) {
                                  if (reason == null) {
                                      reason = ex;
                                  }
                              }
              
                          } else {
                              println("    skipping: " + aDriver.getClass().getName());
                          }
                      } 
                  }
   ```
    
 - MySQL的Driver类
 
  ```java
    // 使用反射就会触发静态块的调用
       public Driver() throws SQLException {
         }   
      static {   
         try {
             DriverManager.registerDriver(new Driver());
         } catch (SQLException var1) {
             throw new RuntimeException("Can't register driver!");
         }
   }
 ```
       
- 多种实现
 
 ``` 
  JDBC为不同的数据库提供了相同的接口Connection,如果加入不同的驱动，那么就会有Connection接口的不同实现。
 ```      
  
  
  ### 桥接模式的提取
         
          
 #### 抽象部分
 ​
  > DriverManager 作为抽象化，它引入了Driver这个实现化，它会把获取Connection行为委托给Driver来执行。
 ``` java
  // DirverInfo 对Driver进行了简单的封装
  CopyOnWriteArrayList<DriverInfo> registeredDrivers = new CopyOnWriteArrayList<>();
 ```      
  
 #### 实现部分
   
   不同的数据库驱动的具体实现作为实现化，DriverManager中引入了DriverInfo（Driver）,getConnection方法间接调用了Driver的connect方法，也就是说把获取连接的行为
   委托给了Driver的具体实现。以后，对于多种数据库驱动就可以任意扩展。
   
   ``` java
    // getConnection方法，使用引入进来的Driver来获取数据库连接 
      for(DriverInfo aDriver : registeredDrivers) {
                 if(isDriverAllowed(aDriver.driver, callerCL)) {
                     try {
                         println("    trying " + aDriver.driver.getClass().getName());
                         Connection con = aDriver.driver.connect(url, info);
                         if (con != null) {
                             // Success!
                             println("getConnection returning " + aDriver.driver.getClass().getName());
                             return (con);
                         }
                     } catch (SQLException ex) {
                         if (reason == null) {
                             reason = ex;
                         }
                     }
                 } else {
                     println("    skipping: " + aDriver.getClass().getName());
                 }
             }
   ``` 
   

   ​

   ​

   ​

   ​

   ​



