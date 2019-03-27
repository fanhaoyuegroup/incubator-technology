## `dubbo-Directory源码总结`

### 1.1 简介

- `服务目录`中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息,比如 ip、端口、服务协议等。

- 在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录

- 如果这样说，` 服务目录 ` 和 ` 注册中心 `的功能不就雷同了吗？确实如此，这里这么说是为了方便大家理解。实际上服务目录在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务目录最终持有的对象

### 1.2 继承体系

- 服务目录目前内置的实现有两个，分别为 StaticDirectory 和 RegistryDirectory，它们均是 AbstractDirectory 的子类。AbstractDirectory 实现了 Directory 接口，这个接口包含了一个重要的方法定义，即 list(Invocation)，用于列举 Invoker

- ![image](http://dubbo.apache.org/docs/zh-cn/source_code_guide/sources/images/directory-inherit-hierarchy.png)

- 如上，Directory 继承自 Node 接口，Node 这个接口继承者比较多，像 Registry、Monitor、Invoker 等均继承了这个接口。这个接口包含了一个获取配置信息的方法 getUrl，实现该接口的类可以向外提供配置信息。另外，大家注意看 RegistryDirectory 实现了 NotifyListener 接口，当注册中心节点信息发生变化后，RegistryDirectory 可以通过此接口方法得到变更信息，并根据变更信息动态调整内部 Invoker 列表。

### 1.3 AbstractDirectory

- AbstractDirectory 封装了 Invoker 列举流程，具体的列举逻辑则由子类实现，这是典型的模板模式。

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed...");
    }
    
    // 调用 doList 方法列举 Invoker，doList 是模板方法，由子类实现
    List<Invoker<T>> invokers = doList(invocation);
    
    // 获取路由 Router 列表
    List<Router> localRouters = this.routers;
    if (localRouters != null && !localRouters.isEmpty()) {
        for (Router router : localRouters) {
            try {
                // 获取 runtime 参数，并根据参数决定是否进行路由
                if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
                    // 进行服务路由
                    invokers = router.route(invokers, getConsumerUrl(), invocation);
                }
            } catch (Throwable t) {
                logger.error("Failed to execute router: ...");
            }
        }
    }
    return invokers;
}

// 模板方法，由子类实现
protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;
```

### 1.4 StaticDirectory

- StaticDirectory 即静态服务目录，顾名思义，它内部存放的 Invoker 是不会变动的。所以，理论上它和不可变 List 的功能很相似。
```java
public class StaticDirectory<T> extends AbstractDirectory<T> {

    // Invoker 列表
    private final List<Invoker<T>> invokers;
    
    // 省略构造方法

    @Override
    public Class<T> getInterface() {
        // 获取接口类
        return invokers.get(0).getInterface();
    }
    
    // 检测服务目录是否可用
    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        for (Invoker<T> invoker : invokers) {
            if (invoker.isAvailable()) {
                // 只要有一个 Invoker 是可用的，就认为当前目录是可用的
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        if (isDestroyed()) {
            return;
        }
        // 调用父类销毁逻辑
        super.destroy();
        // 遍历 Invoker 列表，并执行相应的销毁逻辑
        for (Invoker<T> invoker : invokers) {
            invoker.destroy();
        }
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        // 列举 Inovker，也就是直接返回 invokers 成员变量
        return invokers;
    }
}
```

### 1.5 RegistryDirectory

- RegistryDirectory 是一种动态服务目录，实现了 NotifyListener 接口。当注册中心服务配置发生变化后，RegistryDirectory 可收到与当前服务相关的变化。收到变更通知后，RegistryDirectory 可根据配置变更信息刷新 Invoker 列表。

- Invoker 列举逻辑封装在 doList 方法中，相关代码如下：

```java
public List<Invoker<T>> doList(Invocation invocation) {
    if (forbidden) {
        // 服务提供者关闭或禁用了服务，此时抛出 No provider 异常
        throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
            "No provider available from registry ...");
    }
    List<Invoker<T>> invokers = null;
    // 获取 Invoker 本地缓存
    Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap;
    if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
        // 获取方法名和参数列表
        String methodName = RpcUtils.getMethodName(invocation);
        Object[] args = RpcUtils.getArguments(invocation);
        // 检测参数列表的第一个参数是否为 String 或 enum 类型
        if (args != null && args.length > 0 && args[0] != null
                && (args[0] instanceof String || args[0].getClass().isEnum())) {
            // 通过 方法名 + 第一个参数名称 查询 Invoker 列表，具体的使用场景暂时没想到
            invokers = localMethodInvokerMap.get(methodName + "." + args[0]);
        }
        if (invokers == null) {
            // 通过方法名获取 Invoker 列表
            invokers = localMethodInvokerMap.get(methodName);
        }
        if (invokers == null) {
            // 通过星号 * 获取 Invoker 列表
            invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
        }
        
        // 冗余逻辑，pull request #2861 移除了下面的 if 分支代码
        if (invokers == null) {
            Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
            if (iterator.hasNext()) {
                invokers = iterator.next();
            }
        }
    }
	// 返回 Invoker 列表
    return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
}
```

- #Registry#doRefer 有一个代码subscribeUrl(args)
```java
 private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
            directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        
        directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```

- #RegistryDirectory#subscribe 订阅
```java
public void subscribe(URL url) {
        setConsumerUrl(url);
        consumerConfigurationListener.addNotifyListener(this);
        serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
        registry.subscribe(url, this);
    }
```

- #RegistryDirectory#notify 通知
```java
public synchronized void notify(List<URL> urls) {
        // 过滤分组拿到不同URL列表
        // 从url中获取协议字段，例如condition://、route://、script://、override://等。
        // 获取url的category,在注册中心的命令空间，例如:providers、configurators、routers。
        // 如果category等于routers或协议等于route，则添加到routerUrls中。
        // 如果category等于configurators或协议等于override，则添加到configuratorUrls中。
        // 如果category等于providers，则表示服务提供者url，加入到invokerUrls中。
        Map<String, List<URL>> categoryUrls = urls.stream()
                .filter(Objects::nonNull)
                .filter(this::isValidCategory)
                .filter(this::isNotCompatibleFor26x)
                .collect(Collectors.groupingBy(url -> {
                    if (UrlUtils.isConfigurator(url)) {
                        return CONFIGURATORS_CATEGORY;
                    } else if (UrlUtils.isRoute(url)) {
                        return ROUTERS_CATEGORY;
                    } else if (UrlUtils.isProvider(url)) {
                        return PROVIDERS_CATEGORY;
                    }
                    return "";
                }));

        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);

        // providers 刷新提供者
        List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
        refreshOverrideAndInvoker(providerURLs);
    }
```

- #RegistryDirectory#refreshOverrideAndInvoker(args)
```java
private void refreshInvoker(List<URL> invokerUrls) {
        Assert.notNull(invokerUrls, "invokerUrls should not be null");
        //如果invokerUrls不为空并且长度为1，并且协议为empty,表示该服务的所有服务提供者都下线了。需要销毁当前所有的服务提供者Invoker。
        if (invokerUrls.size() == 1
                && invokerUrls.get(0) != null
                && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            this.forbidden = true; // Forbid to access
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            destroyAllInvokers(); // Close all invokers
        } else {
            this.forbidden = false; // Allow to access
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            /*如果invokerUrls为空，并且已缓存的invokerUrls不为空，将缓存中的invoker url复制到invokerUrls中，这里可以说明如果providers目录未发送变化，invokerUrls则为空，表示使用上次缓存的服务提供者URL对应的invoker；如果invokerUrls不为空，则用iinvokerUrls中的值替换原缓存的invokerUrls，这里说明，如果providers发生变化，invokerUrls中会包含此时注册中心所有的服务提供者。如果invokerUrls为空，则无需处理，结束本次更新服务提供者Invoker操作。*/
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            /**
             * If the calculation is wrong, it is not processed.
             *
             * 1. The protocol configured by the client is inconsistent with the protocol of the server.
             *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
             * 2. The registration center is not robust and pushes illegal specification data.
             *
             */
            if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                        .toString()));
                return;
            }

            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
            // pre-route and build cache, notice that route cache should build on original Invoker list.
            // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
            routerChain.setInvokers(newInvokers);
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

            try {
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```




