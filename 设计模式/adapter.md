## adapter模式（也称wrapper模式）
##### 现实中的案例：
    1.日本住宅区常用电压是100伏特，而国内是220伏特，此时一个人去日本出差,那么这个人怎么让手机充电呢
    2.于是为了解决这不同国家住宅常用电压之间尴尬的问题，开发了适配器
![图片Alt](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1542965187129&di=372e61611db3471357c994f8297ba523&imgtype=0&src=http%3A%2F%2Fs3.51cto.com%2Fwyfs02%2FM00%2F12%2F4A%2FwKiom1MB7T3x_RE4AAROTBX8dOk215.jpg "图片Title")

##1.介绍
1. 意图：将一个类的接口转换成客户希望的另外一个接口。适配器模式使得原本由于接口不兼容而不能一起工作的那些类可以一起工作

2. 主要解决：主要解决在软件系统中，常常要将一些"现存的对象"放到新的环境中，而新环境要求的接口是现对象不能满足的。

3. 如何解决：继承或依赖（推荐）。

4. 类适配器模式、对象适配模式

4. 关键代码：适配器继承或依赖已有的对象，实现想要的目标接口。

5. 优点： 
    - 兼容性高。 
    - 提高了类的复用。 
    - 灵活性好。
6. 缺点：
    -  过多地使用适配器，会让系统非常零乱，不易整体进行把握。比如，明明看到调用的是 A 接口，其实内部被适配成了 B 接口的实现，一个系统如果太多出现这种情况，无异于一场灾难。因此如果不是很有必要，可以不使用适配器，而是直接对系统进行重构。 2.由于 JAVA 至多继承一个类，所以至多只能适配一个适配者类，而且目标类必须是抽象类。 
7. 注意事项：适配器不是在详细设计时添加的，而是解决正在服役的项目的问题。
##2.案例
1. client(电脑) -> 使用(adapter)适配器 -> 适配器(adapter)需要实现原有的接口，并添加转换方法  

2. 代码见design-pattern模块-adapter（类适配模式）

3. 对象适配模式只是把目标设计为抽象类，然后子类集成抽象类实现适配方法，然后子类内部设置一个原有接口实现类

##3.Spring使用适配器模式干了啥（对比我们之前的写法）
1. SpringMVC使用适配器，首先我们看一张SpringMVC-Controller类图(adaptee)
2. ![](https://img-blog.csdn.net/20161223100407771?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDI4ODI2NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
3. 可以看出，SpringMVC在对我们定义的Controller类做了不同处理，然后我们再看下DispatcherServlet流程
```java
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
            HttpServletRequest processedRequest = request;
            HandlerExecutionChain mappedHandler = null;
            boolean multipartRequestParsed = false;
     
            WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
     
            try {
                ModelAndView mv = null;
                Exception dispatchException = null;
     
                try {
                    processedRequest = checkMultipart(request);
                    multipartRequestParsed = (processedRequest != request);
     
                    // 此处通过HandlerMapping来映射Controller
                    mappedHandler = getHandler(processedRequest);
                    if (mappedHandler == null || mappedHandler.getHandler() == null) {
                        noHandlerFound(processedRequest, response);
                        return;
                    }
     
                    // 获取适配器
                    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
     
                    // Process last-modified header, if supported by the handler.
                    String method = request.getMethod();
                    boolean isGet = "GET".equals(method);
                    if (isGet || "HEAD".equals(method)) {
                        long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                        if (logger.isDebugEnabled()) {
                            logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                        }
                        if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                            return;
                        }
                    }
     
                    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                        return;
                    }
     
                    // 通过适配器调用controller的方法并返回ModelAndView
                    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
     
                    if (asyncManager.isConcurrentHandlingStarted()) {
                        return;
                    }
     
                    applyDefaultViewName(processedRequest, mv);
                    mappedHandler.applyPostHandle(processedRequest, response, mv);
                }
                catch (Exception ex) {
                    dispatchException = ex;
                }
                catch (Throwable err) {
                    // As of 4.3, we're processing Errors thrown from handler methods as well,
                    // making them available for @ExceptionHandler methods and other scenarios.
                    dispatchException = new NestedServletException("Handler dispatch failed", err);
                }
                processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
            }
            catch (Exception ex) {
                triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
            }
            catch (Throwable err) {
                triggerAfterCompletion(processedRequest, response, mappedHandler,
                        new NestedServletException("Handler processing failed", err));
            }
            finally {
                if (asyncManager.isConcurrentHandlingStarted()) {
                    // Instead of postHandle and afterCompletion
                    if (mappedHandler != null) {
                        mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                    }
                }
                else {
                    // Clean up any resources used by a multipart request.
                    if (multipartRequestParsed) {
                        cleanupMultipart(processedRequest);
                    }
                }
            }
	}
```
- 那如果假设没有适配器模式，那么DispatchServlet#doDispatch流程会多很多if...else...判断controller类型
- Spring如何解决的呢?
1. 首先创建一个适配器接口(角色-target)
```java
public interface HandlerAdapter {
    
	boolean supports(Object handler);
	
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
}
```
适配器实现类（角色-adapter）
![](https://img-blog.csdn.net/20161223103206471?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDI4ODI2NA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

DispatchServlet#doDispatch部分代码(相当于client角色)
```markdown
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

1.getHandlerAdapter()方法 传入原始的controller的handler(相当于需要被转换者)，然后返回适配器类
2.然后使用适配器的handle方法处理
``` 

##总结
1. 适配器模式的角色client(客户)->adapter(使用适配器)->target(被实现的东西)->adaptee(老接口)
2. 比较Spring使用适配器模式，自己设计的不够优雅，可以采取这种类设计
3. 能解决不改变原有接口，兼容老接口，开发新的功能需求

##文档相关链接
1. 从SpringMVC来看适配器模式: https://blog.csdn.net/u010288264/article/details/53835185
2. 解析SpringMVC源码中使用到的“适配器”模式: https://blog.csdn.net/w1033162186/article/details/50635348
3. Spring中的设计模式-适配器模式: https://blog.csdn.net/adoocoke/article/details/8286902