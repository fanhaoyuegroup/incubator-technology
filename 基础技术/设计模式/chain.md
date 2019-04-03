## 责任链

### 一、`概念`
1. 将一个业务处理流程请求形成一条链式，并且这条链上的处理者能处理则处理，不能处理则往后推
2. 这个思想来源于团队协作。例如：消防群的消息发出，而服务端的所有人就是责任者，看到者如果能处理则处理，不能处理就交给其他人处理。

### 二、`为什么会用到这种模式`
1. 可以解决业务上大量if-else判断的问题，将if-else里面的逻辑抽离成一个类或者方法，符合面向对象编程
2. 可以快速的植入责任者进行处理业务，独立性更强，有点类似 `插拔式` 编程一样

### 三、`可能存在的问题`
1. 推卸请求会导致处理延迟吗？（提问）

### 四、`UML设计图`
![image](https://upload-images.jianshu.io/upload_images/1234352-e34f882565fe16ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/539)
- 这种是每个实现类都setNext指向下一个实现类

![image](https://static.oschina.net/uploads/img/201709/11101627_hx5w.png)
- 这个设计方案采取web容器责任链模式设计

### 五、`应用场景`
1. filter
2. 拦截器
3. if-else繁多场景
4. ...

### 六、`示例`
[责任链模式代码demo](https://github.com/fanhaoyuegroup/interest-group/tree/master/design-pattern/src/main/java/com/fan/design/chain)