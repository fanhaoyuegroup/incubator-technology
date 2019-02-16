#dubbo配置初始化

**dubbo源码 (https://github.com/apache/incubator-dubbo)**
>* 官方推荐zookeeper作为注册中心，先安装部署zk
>* zookeeper下载地址：https://archive.apache.org/dist/zookeeper/
>* 自行下载安装部署，然后到 bin目录启动 ./zkServer.sh start



**dubbo服务暴露与引用主要做了三件事**
>* 注册中心启动，监听消息提供者的注册服务、接收消息消费者的服务订阅（服务注册与发现机制）。
>* 服务提供者向注册中心注册服务。
>* 服务消费者向注册中心订阅服务


**源码提供的服务配置**
``` 
  <!--服务提供者的应用名 用来计算依赖关系 -->
    <dubbo:application name="demo-provider"/>

    <!-- 用zk做服务的注册中心 -->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>

    <!-- 用dubbo协议 通过20880暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- 服务的具体实现，跟本地的bean一样 -->
    <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl"/>

    <!-- 服务被暴露的接口 -->
    <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService"/>
```

Spring自定义标签加载dubbo配置：dubbo自定义标签与命名空间其实现代码在模块dubbo-config中。

DubboNamespaceHandler继承NamespaceHandlerSupport实现自定义标签扩展。

自定义标签主要有 ：
application、module、registry、monitor、provider、consumer、protocol、service、reference、annotation
```
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        //注册配置解析器
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
```

dubbo提供了来年规划中配置解析器，对应两种配置方式：
DubboBeanDefinitionParser(基于xml配置文件)、AnnotationBeanDefinitionParser(基于注解)。
   
解析器就是将Element element(xml节点)解析成BeanDefinition。

在dubbo-config-spring模块下，src/main/resouce/META-INF的配置文件dubbo.xsd、spring.handlers、spring.schemas，
基于spring的SPI实现dubbo自定义的配置。

最后，完成dubbo的配置解析与实例化。


