# canal源码分析 (三) 内嵌服务器启动
## 引言
####本章我们将进入到canal的内嵌服务器CanalServerWithEmbedded启动过程，通过这个类，这个类可以看做canal内部黑盒处理对外暴露的一些。在CanalServerWithEmbedded这个类中，我们可以看到有两个start的函数方法。从之前的文章中，我们已经已经看到了这些方法的具体方法。所以本文就是直接关注这个类内部的一些启动逻辑

## 入口函数
#### start()方法
```
    public void start() {
    	// 若内嵌服务器未启动
        if (!isStart()) {
            super.start();
            // 如果存在provider,则启动metrics service
            // 接下来三行都是关于开启一些普罗米修斯监控接口的方法
            loadCanalMetrics();
            metrics.setServerPort(metricsPort);
            metrics.initialize();
            // 然后将所有的canalInstance都存放在一个map中，并用之前的instance生成器生成新的canalInstance
            canalInstances = MigrateMap.makeComputingMap(new Function<String, CanalInstance>() {

                public CanalInstance apply(String destination) {
                    return canalInstanceGenerator.generate(destination);
                }
            });

            // lastRollbackPostions = new MapMaker().makeMap();
        }
    }
```
### 从这个方法中也可以看到，这个start()方法，只是做了一些初始化的工作，所以重点看一下另外一个start方法
#### start(final String destination)
```
    public void start(final String destination) {
    	// 通过destination名字从map中获取canalInstance，若不存在则用canalInstance生成器创造
        final CanalInstance canalInstance = canalInstances.get(destination);
        // 若得到的canalInstance还未启动
        if (!canalInstance.isStart()) {
            try {
                MDC.put("destination", destination);
                // 在普罗米修斯的状态中注册
                if (metrics.isRunning()) {
                    metrics.register(canalInstance);
                }
                // 启动这个canalInstance，从这句开始拉开了所有事情的帷幕
                canalInstance.start();
                logger.info("start CanalInstances[{}] successfully", destination);
            } finally {
                MDC.remove("destination");
            }
        }
    }
```
#### 从方法的入参里也可以看到两个start方法的区别。传入了destination，所以就是会去启动相应的canalInstance的实例。也就是可以看做这个类作为所有真正实例的统一入口，所有的操作都由这个类进行统一代理。