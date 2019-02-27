# canal源码分析 (二) 启动过程
## 引言
#### 上一章导读中我们算是初步对canal这个中间件的代码结构有了一些大概的印象，我也对其中我觉得比较关键的模块进行了一些浅显的论述，从本章开始，我们要真正开始对各个模块进行详细的分析。好了，进入正题。本章会详细论述一下canal是如何启动的以及启动的过程中做了什么。

## 启动入口
#### 看完上一章的同学应该也可以看到在canal的代码结构中有一个canal.deployer的模块，从名字也可以看出来这就是canal的启动器所在的模块。在里面找到CanalLauncher这个类。作者在这个类的注释中也写的很清楚
> /**
> * canal独立版本启动的入口类
> */ >

#### 好的，既然作者都这么说了。直接运行这个类里的main函数。大功告成！？ 诶不对，好像报错了。果然还有点小问题还没有解决。看看报错内容
``` java
	2019-01-22 01:21:50.454 [destination = example , address = /127.0.0.1:3306 , EventParser] ERROR c.a.o.c.p.inbound.mysql.rds.RdsBinlogEventParserProxy - dump address /127.0.0.1:3306 has an error, retrying. caused by 
com.alibaba.otter.canal.parse.exception.CanalParseException: java.io.IOException: connect /127.0.0.1:3306 failure
Caused by: java.io.IOException: connect /127.0.0.1:3306 failure
	at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.connect(MysqlConnector.java:77) ~[classes/:na]
	at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.connect(MysqlConnection.java:89) ~[classes/:na]
	at com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.preDump(MysqlEventParser.java:86) ~[classes/:na]
	at com.alibaba.otter.canal.parse.inbound.AbstractEventParser$3.run(AbstractEventParser.java:175) ~[classes/:na]
	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_111]
```
###### 注意：如果你本机确实启动了一个数据库服务，那么是不会报错的。

#### 看一下这个报错内容给了我们什么信息
 - 是 "destination = example" 的这个canal.instance 出错了 (还记得在上一章中我说过destination和canal.instance会在后面一直出现嘛...)
 - 报错原因是因为代码去连接本机的3306端口时候报错
 
#### 看到这两个得到的信息，我们再进一步提取出我们解决这个问题的思路。肯定是canal启动的时候，根据哪些配置或者默认的参数生成了默认为example的canal.instance然后直接监听了本机启动的数据库。所以只要找到这个配置的地方，然后把他修改成我们真正需要监听的数据库地址，就可以解决这个报错了。所以就是这个启动类是正确的，但是某些配置没有配置成功。这边先卖个关子，等我们看完整个启动过程，你们肯定会知道在哪里可以修改这个配置，让启动顺利完成。

## 启动详解
#### 回到我们刚才的main函数中,去掉前面的一些打印日志,真正起作用的第一部分是下面的这几行代码

``` java
			// 获取canal启动的配置文件的目录，提供默认实现方式
			// 默认的配置文件可以看resource目录下的canal.properties文件
            String conf = System.getProperty("canal.conf", "classpath:canal.properties");
            Properties properties = new Properties();
            ManagerRemoteConfigMonitor managerDbConfigMonitor = null;
            
            // 根据之前获得的配置文件目录去实例化Properties类
            if (conf.startsWith(CLASSPATH_URL_PREFIX)) {
                conf = StringUtils.substringAfter(conf, CLASSPATH_URL_PREFIX);
                properties.load(CanalLauncher.class.getClassLoader().getResourceAsStream(conf));
            } else {
                properties.load(new FileInputStream(conf));
            }
            
            // 在配置中若注明了以下参数，表名canal的启动配置从远端获取
            String jdbcUrl = properties.getProperty("canal.manager.jdbc.url");
            if (!StringUtils.isEmpty(jdbcUrl)) {
                logger.info("## load remote canal configurations");
                // load remote config
                String jdbcUsername = properties.getProperty("canal.manager.jdbc.username");
                String jdbcPassword = properties.getProperty("canal.manager.jdbc.password");
                
                // 内部实现的逻辑也就是根据提供的数据库信息，去数据库中获取内容，并覆盖本地配置文件
                // 从数据库中获取信息的的sql语句
                // "select name, content, modified_time from canal_config where id=1"
                
                managerDbConfigMonitor = new ManagerRemoteConfigMonitor(jdbcUrl, jdbcUsername, jdbcPassword);
                // 加载远程canal.properties
                Properties remoteConfig = managerDbConfigMonitor.loadRemoteConfig();
                // 加载remote instance配置
                managerDbConfigMonitor.loadRemoteInstanceConfigs();
                if (remoteConfig != null) {
                    properties = remoteConfig;
                } else {
                    managerDbConfigMonitor = null;
                }
            } else {
                logger.info("## load canal configurations");
            }
```
#### 主要的几点我在代码注释里也写了，目的就是从配置文件中得到相应的配置信息，作为后续启动时的一些参数。再看接下来的代码。

``` java
		final CanalStater canalStater = new CanalStater();
		canalStater.start(properties);
```
#### 怎么在canalLancer里又new了一个canalStart，还把之前生成的properties作为他的start方法的参数传入。看了一下这个类的构造函数是空的，所以直接跟进去这个类的start方法，看看做了什么。

``` java
		// 从配置类中读取Server的启动模式，默认为tcp，还提供了另外两种模式 kafka，rocketmq
		String serverMode = CanalController.getProperty(properties, CanalConstants.CANAL_SERVER_MODE);
        if (serverMode.equalsIgnoreCase("kafka")) {
            canalMQProducer = new CanalKafkaProducer();
        } else if (serverMode.equalsIgnoreCase("rocketmq")) {
            canalMQProducer = new CanalRocketMQProducer();
        }

        if (canalMQProducer != null) {
            // disable netty
            System.setProperty(CanalConstants.CANAL_WITHOUT_NETTY, "true");
            System.setProperty(CanalConstants.CANAL_DESTINATIONS,
                properties.getProperty(CanalConstants.CANAL_DESTINATIONS));
        }
```
#### 这一段代码初步看起来也不知道做了什么，就是根据配置来选择服务端启动方式。然后如果是mq的模式，就禁用之后的netty操作。目前还不知道生成的CanalProducer类具体是干嘛的，也不知道System.setProperty两个属性之后起到什么作用。先不管了，接着往后看。
``` java
        controller = new CanalController(properties);
        controller.start();
```
#### 嗯？怎么又来了一个CanalController，简直要被绕晕了。继续看这个类的构造函数

``` java
        managerClients = MigrateMap.makeComputingMap(new Function<String, CanalConfigClient>() {

            public CanalConfigClient apply(String managerAddress) {
                return getManagerClient(managerAddress);
            }
        });
        
        // 初始化全局参数设置
        globalInstanceConfig = initGlobalConfig(properties);
```
#### 开局一个map，不知道存了什么东西。先跳过了。然后初始化全局参数设置是什么呢。再跟进去看看这个方法。
``` java
InstanceConfig globalConfig = new InstanceConfig();
        String modeStr = getProperty(properties, CanalConstants.getInstanceModeKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(modeStr)) {
            globalConfig.setMode(InstanceMode.valueOf(StringUtils.upperCase(modeStr)));
        }

        String lazyStr = getProperty(properties, CanalConstants.getInstancLazyKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(lazyStr)) {
            globalConfig.setLazy(Boolean.valueOf(lazyStr));
        }

        String managerAddress = getProperty(properties,
            CanalConstants.getInstanceManagerAddressKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(managerAddress)) {
            globalConfig.setManagerAddress(managerAddress);
        }

        String springXml = getProperty(properties, CanalConstants.getInstancSpringXmlKey(CanalConstants.GLOBAL_NAME));
        if (StringUtils.isNotEmpty(springXml)) {
            globalConfig.setSpringXml(springXml);
        }

        instanceGenerator = new CanalInstanceGenerator() {

            public CanalInstance generate(String destination) {
          			...
            }

        };

        return globalConfig;   
```
#### 整个方法的大逻辑就是生成并返回一个globalConfig实例，并把配置文件中的一些配置设置为globalConfig的成员变量。具体这些参数是什么意思放在后面遇到的时候再讲。主要看到最后的那个instanceGenerator
``` java
		// 如名字所说 这个类是canalInstance的生成器，内部实现一个generate方法
		instanceGenerator = new CanalInstanceGenerator() {
            public CanalInstance generate(String destination) {
                // 下文会讲到这个map, 存放每个instance独立的配置，先占个坑
                InstanceConfig config = instanceConfigs.get(destination);
                if (config == null) {
                    throw new CanalServerException("can't find destination:{}");
                }
				// 根据获取的配置的生成模式选择具体的instance生成器的实现类
				// 目前有两种实现方式 manager和spring
                if (config.getMode().isManager()) {
                    ManagerCanalInstanceGenerator instanceGenerator = new ManagerCanalInstanceGenerator();
                    // 会用到我们之前提到的那个开局一个map地方instanceGenerator.setCanalConfigClient(managerClients.get(config.getManagerAddress()));
                    return instanceGenerator.generate(destination);
                } else if (config.getMode().isSpring()) {
                    SpringCanalInstanceGenerator instanceGenerator = new SpringCanalInstanceGenerator();
                    // 默认的配置是spring模式的
                    // 为什么要加锁？
                    synchronized (CanalEventParser.class) {
                        try {
                            // 设置当前正在加载的通道，加载spring查找文件时会用到该变量
                            System.setProperty(CanalConstants.CANAL_DESTINATION_PROPERTY, destination);
                            // 默认路径 classpath:spring/file-instance.xml
                            instanceGenerator.setBeanFactory(getBeanFactory(config.getSpringXml()));
                            // 用配置的spring.xml文件来生成canalInstance bean
                            return instanceGenerator.generate(destination);
                        } catch (Throwable e) {
                            logger.error("generator instance failed.", e);
                            throw new CanalException(e);
                        } finally {
                            System.setProperty(CanalConstants.CANAL_DESTINATION_PROPERTY, "");
                        }
                    }
                } else {
                    throw new UnsupportedOperationException("unknow mode :" + config.getMode());
                }

            }

        };
```
#### 这边罗列一下spring.xml文件中配置
``` xml
	<bean id="instance" class="com.alibaba.otter.canal.instance.spring.CanalInstanceWithSpring">
		<property name="destination" value="${canal.instance.destination}" />
		<property name="eventParser">
			<ref local="eventParser" />
		</property>
		<property name="eventSink">
			<ref local="eventSink" />
		</property>
		<property name="eventStore">
			<ref local="eventStore" />
		</property>
		<property name="metaManager">
			<ref local="metaManager" />
		</property>
		<property name="alarmHandler">
			<ref local="alarmHandler" />
		</property>
        <property name="mqConfig">
            <ref local="mqConfig" />
        </property>
	</bean>
```
#### 可以看到这个instance里面的几个重要的组件，跟我们第一章中对代码结构分析时提到的几个重要组件是一样的，所以印证了我之前说的
	canalInstance可以看成对mysql的binlog文件进行解析存储处理的容器通道
#### 好了，结束了这个方法，再看调用这个方法之后又做了什么
```
		// 和之前的初始化全局配置差不多，这部分是初始化每个单独的instance的配置，没有的话就默认全局配置
        instanceConfigs = new MapMaker().makeMap();
        // 初始化instance config
        initInstanceConfig(properties); 
        // init socketChannel
        String socketChannel = getProperty(properties, CanalConstants.CANAL_SOCKETCHANNEL);
        if (StringUtils.isNotEmpty(socketChannel)) {
            System.setProperty(CanalConstants.CANAL_SOCKETCHANNEL, socketChannel);
        }

        // 兼容1.1.0版本的ak/sk参数名
        String accesskey = getProperty(properties, "canal.instance.rds.accesskey");
        String secretkey = getProperty(properties, "canal.instance.rds.secretkey");
        if (StringUtils.isNotEmpty(accesskey)) {
            System.setProperty(CanalConstants.CANAL_ALIYUN_ACCESSKEY, accesskey);
        }
        if (StringUtils.isNotEmpty(secretkey)) {
            System.setProperty(CanalConstants.CANAL_ALIYUN_SECRETKEY, secretkey);
        }
```

# TODO parseInstanceConfig
#### 主要也就是讲了一下从配置里读取信息的内容，这边也不做赘述了。直接看下面这段比较重要的部分
```
		// 准备canal server
        cid = Long.valueOf(getProperty(properties, CanalConstants.CANAL_ID));
        ip = getProperty(properties, CanalConstants.CANAL_IP);
        port = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_PORT));
        // 得到单例的内嵌canal服务器
        embededCanalServer = CanalServerWithEmbedded.instance();
        // 并把之前生成的canalInstance生成器作为成员变量
        embededCanalServer.setCanalInstanceGenerator(instanceGenerator);// 设置自定义的instanceGenerator
        // 提供给普罗米修斯的监控端口
        try {
            int metricsPort = Integer.valueOf(getProperty(properties, CanalConstants.CANAL_METRICS_PULL_PORT));
            embededCanalServer.setMetricsPort(metricsPort);
        } catch (NumberFormatException e) {
            logger.info("No valid metrics server port found, use default 11112.");
            embededCanalServer.setMetricsPort(11112);
        } 
```
#### 这边启动的embededCanalServer是一种内嵌式的canal启动服务器的实现方式，里面会保存所有的canalInstance。且同一个jvm只存在一个embededCanalServer。然后暴露metrics端口给普罗米修斯拉取监控数据。再往下看
```
		// 还记得之前有一个步骤判断了一下是哪种canalServer的地方吗,tcp . rocketmq . kafka三种模式的地方
		String canalWithoutNetty = getProperty(properties, CanalConstants.CANAL_WITHOUT_NETTY);
		// 若采用的是传统的tcp的方式，也就是netty连接,则会得到CanalServerWithNetty的实例化对象，并设置ip和端口
        if (canalWithoutNetty == null || "false".equals(canalWithoutNetty)) {
            canalServer = CanalServerWithNetty.instance();
            canalServer.setIp(ip);
            canalServer.setPort(port);
        }
```
#### 这里提一下这两段代码中的embededCanalServer和canalServer是什么关系，或者说有什么区别
	- embededCanalServer是在程序中启动的，内部保存了所有canalInstance实例的，可以看做是所有同步数据库数据的通道的第一个启动入口
	- canalServer是一个在代码中暴露出的一个服务端的接口，如果是默认的netty实现的话，就是暴露一个网络接口供客户端来网络请求的
#### 好的，回到代码中。接下来的几行代码主要就是如果给定了zk地址，就要在zk里搞点事情。创建两个永久的zk node (/otter/canal/destinations和/otter/canal/cluster)，都是用作之后的的元数据存储的路径，也先不说了，之后在讲到canal.meta这个模块的时候也会提到。
#### 在构造函数的最后，首先通过调用静态函数的方式，为ServerRunningMonitors类传入了静态成员变量。然后因为默认autoScan这个值是true的，所以又初始化了defaultAction这个方法类和instanceConfigMonitors这个compute map。 
#### 好的，感觉在构造函数里困了太久了，先留一些问题，感觉从这里面出来。回到CanalStarter这个类中，构造完CanalController类之后，紧接着直接调用了他的start方法。
```
        logger.info("## start the canal server[{}:{}]", ip, port);
        // 创建整个canal的工作节点
        final String path = ZookeeperPathUtils.getCanalClusterNode(ip + ":" + port);
        initCid(path);
        if (zkclientx != null) {
            this.zkclientx.subscribeStateChanges(new IZkStateListener() {

                public void handleStateChanged(KeeperState state) throws Exception {

                }

                public void handleNewSession() throws Exception {
                    initCid(path);
                }

                @Override
                public void handleSessionEstablishmentError(Throwable error) throws Exception {
                    logger.error("failed to connect to zookeeper", error);
                }
            });
        } 
```
#### 这部分代码做的还是初始化的工作。若在配置中有填写zk地址，则会在zk的/otter/canal/cluster/路径下创建一个临时节点，标识当前canal机器已经注册并启动了。接下来的一行才是真正的开始我们canal最想要完成的从数据库拉binlog信息并解析存储的所有工作.
```
        // 优先启动embeded服务
        embededCanalServer.start();
```
#### 这个start之后，相当于真正打开了所有通道的开关，之后所有通道里面的数据就会像水流一样源源不断的流进流出。这里面我觉得需要单独开一章讲解，所以这里先跳过，下一章再讲。先继续看下面的部分。
```
        // 尝试启动一下非lazy状态的通道
        for (Map.Entry<String, InstanceConfig> entry : instanceConfigs.entrySet()) {
            final String destination = entry.getKey();
            InstanceConfig config = entry.getValue();
            // 创建destination的工作节点
            if (!embededCanalServer.isStart(destination)) {
                // HA机制启动
                ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
                if (!config.getLazy() && !runningMonitor.isStart()) {
                    runningMonitor.start();
                }
            }

            if (autoScan) {
                instanceConfigMonitors.get(config.getMode()).register(destination, defaultAction);
            }
        }
```
##### 这段代码的意思是将之前放在instanceConfig这个map中的所有key拿出来遍历一遍，判断是否已经启动了，如果没有，则调用ServerRunningMonitor的静态方法。
```
        ServerRunningMonitor runningMonitor = ServerRunningMonitors.getRunningMonitor(destination);
		// ServerRunningMonitors.java
		    public static ServerRunningMonitor getRunningMonitor(String destination) {
        return (ServerRunningMonitor) runningMonitors.get(destination);
    }
```
#### 因为这个map中一开始没有这个destination的key，所以会调用map的apply方法，得到一个ServerRunningMonitor类，(注意这个类中的setListener方法设置了一个listener，在后面的代码会用到)，之后再调用这个获取的ServerRunningMonitor类的start方法，最后会跳到下面的这段代码中。 (这部分代码调用可能有点复杂，建议读者自己调试感受一下)
```
    public synchronized void start() {
        super.start();
        try {
        	// 进入这段代码可以看到就是调用了之前setListener方法里的processStart方法
        	// 方法内部做的操作就是zk的/otter/canal/destinations/{destination}/cluster/{node}路径下创建临时节点
            processStart();
            if (zkClient != null) {
                // 如果需要尽可能释放instance资源，不需要监听running节点，不然即使stop了这台机器，另一台机器立马会start
                String path = ZookeeperPathUtils.getDestinationServerRunning(destination);
                zkClient.subscribeDataChanges(path, dataListener);

                initRunning();
            } else {
                processActiveEnter();// 没有zk，直接启动
            }
        } catch (Exception e) {
            logger.error("start failed", e);
            // 没有正常启动，重置一下状态，避免干扰下一次start
            stop();
        }

    }
```


