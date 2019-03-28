# canal从mysql中拉数据的过程
## 前文
#### 本文主要总结一下canal服务在启动后，怎么拉取mysql的数据并解析供我们服务使用。需要说明一下，其实在代码中，并不只是Mysql这一种消息源的，还包括Rds或者localBinlog。但是由于在项目中用到的大多数都是mysql，所以本文会只针对这一种消息源进行分析。
## 正文
#### 直接看到AbstractEventParser的start方法。这个类是在canalInstance初始化的时候，就默认初始化并启动的。这边提一句，每个canalInstance在初始化的时候，会附带几个组件。分别为
- MetaManager
	内存，zk，周期性zk，本地文件
- EventStore
	内存，文件
- EnentSink
	Group和Entry
- EventParse
	Mysql ， LocalBinlog
- AlarmHandler


#### 今天我们主要来看一下EventParse的部分。
```
	public void start() {
		// 调用全局父类的start方法，也就是设置running这个布尔值为true, 并避免重复启动
        super.start();
        MDC.put("destination", destination);
        // 配置transaction buffer
        // 初始化缓冲队列
        // 默认初始大小为1024，Buffer里面包含了一个数组，这个在后面可以详细说一下
        transactionBuffer.setBufferSize(transactionSize);// 设置buffer大小
     	// 启动过程就是初始化了底层的数组
        transactionBuffer.start();
        // 构造bin log parser
        // 也就是将mysql的binlog消息，解析成我们需要的对应的Entry对象
        // 在项目中只有实现了对binlog的解析组件
        binlogParser = buildParser();// 初始化一下BinLogParser
        binlogParser.start();
    	...
    }
```
#### 以上代码就是在开始流程前，初始化一些存储和解析的组件，重点的东西在后面。在后面的代码中，会构造一个线程，所有具体的逻辑都集中在这个线程中，我们现在来看看这个线程做了什么
```
	parseThread = new Thread(new Runnable() {

            public void run() {
                MDC.put("destination", String.valueOf(destination));
                ErosaConnection erosaConnection = null;
                while (running) {
                		// 若组件还在运行的话，就会不断循环
                    try {
                        // 开始执行replication
                        // 1. 构造Erosa连接
                        // 项目中抽象了一个父类叫做erosaConnection
                        // 这个类的子类实现包括mysql，和localBinlog
                        // 也就是抽象实现，面向接口编程
                        erosaConnection = buildErosaConnection();
						... 
     }
     
     
     // 看一下在子类的MysqlEventParser中，是怎么构建这个connector的
     protected ErosaConnection buildErosaConnection() {
        return buildMysqlConnection(this.runningInfo);
     }
     
     // 可以看到，也就是去通过账号密码以及地址，去得到一个mysql的connector，并随机生成一个slaveId
     private MysqlConnection buildMysqlConnection(AuthenticationInfo runningInfo) {
        MysqlConnection connection = new MysqlConnection(runningInfo.getAddress(),
            runningInfo.getUsername(),
            runningInfo.getPassword(),
            connectionCharsetNumber,
            runningInfo.getDefaultDatabaseName());
        connection.getConnector().setReceiveBufferSize(receiveBufferSize);
        connection.getConnector().setSendBufferSize(sendBufferSize);
        connection.getConnector().setSoTimeout(defaultConnectionTimeoutInSeconds * 1000);
        connection.setCharset(connectionCharset);
        connection.setReceivedBinlogBytes(receivedBinlogBytes);
        // 随机生成slaveId
        if (this.slaveId <= 0) {
            this.slaveId = generateUniqueServerId();
        }
        connection.setSlaveId(this.slaveId);
        return connection;
    }

```
#### 以上就是在线程中，先得到了对应的mysql的连接。在得到对应的连接之后，会进行一些对连接的心跳检查和前期的准备工作。
```
	// 2. 启动一个心跳线程
    startHeartBeat(erosaConnection);
       
    protected void startHeartBeat(ErosaConnection connection) {
        lastEntryTime = 0L; // 初始化
        if (timer == null) {// lazy初始化一下
            String name = String.format("destination = %s , address = %s , HeartBeatTimeTask",
                destination,
                runningInfo == null ? null : runningInfo.getAddress().toString());
            synchronized (AbstractEventParser.class) {
                // synchronized (MysqlEventParser.class) {
                // why use MysqlEventParser.class, u know, MysqlEventParser is
                // the child class 4 AbstractEventParser,
                // do this is ...
                // 懒汉双重检查的单例模式，创建一个Timer
                if (timer == null) {
                    timer = new Timer(name, true);
                }
            }
        }

        if (heartBeatTimerTask == null) {// fixed issue #56，避免重复创建heartbeat线程
        	// 构建定时的心跳任务，周期性执行
            heartBeatTimerTask = buildHeartBeatTimeTask(connection);
            Integer interval = detectingIntervalInSeconds;
            timer.schedule(heartBeatTimerTask, interval * 1000L, interval * 1000L);
            logger.info("start heart beat.... ");
        }
    }
    
    protected TimerTask buildHeartBeatTimeTask(ErosaConnection connection) {
        return new TimerTask() {

            public void run() {
                try {
                    if (exception == null || lastEntryTime > 0) {
                        // 如果未出现异常，或者有第一条正常数据
                        long now = System.currentTimeMillis();
                        long inteval = (now - lastEntryTime) / 1000;
                        if (inteval >= detectingIntervalInSeconds) {
                        	// 构建一个心跳包
                            Header.Builder headerBuilder = Header.newBuilder();
                            headerBuilder.setExecuteTime(now);
                            Entry.Builder entryBuilder = Entry.newBuilder();
                            entryBuilder.setHeader(headerBuilder.build());
                            entryBuilder.setEntryType(EntryType.HEARTBEAT);
                            Entry entry = entryBuilder.build();
                            // 提交到sink中，目前不会提交到store中，会在sink中进行忽略
                            consumeTheEventAndProfilingIfNecessary(Arrays.asList(entry));
                        }
                    }

                } catch (Throwable e) {
                    logger.warn("heartBeat run failed ", e);
                }
            }

        };
    }
```
#### 上面的代码也不是什么关键的部分，简单过一下就可以。整个的准备工作就是建立连接，然后做一些前期准备工作。这部分不细说了。直接看到关键的部分。从mysql中拉数据的时候，先要知道是从哪里开始拉取。所以要先得到相应的position。
```
		// 4. 获取最后的位置信息
	    EntryPosition position = findStartPosition(erosaConnection);
	    final EntryPosition startPosition = position;
	    if (startPosition == null) {
	        throw new PositionNotFoundException("can't find start position for " + destination);
	    }
	
	    if (!processTableMeta(startPosition)) {
	        throw new CanalParseException("can't find init table meta for " + destination
	                                      + " with position : " + startPosition);
	    }
	    // 重新链接，因为在找position过程中可能有状态，需要断开后重建
	    erosaConnection.reconnect();
		
		
		
	protected EntryPosition findStartPosition(ErosaConnection connection) throws IOException {
		// 位点有两种模式，分别问Gtid模式和fileName+position
		// 在目前的公司中使用中是尽量使用Gtid，因为位点的回滚更直观
        if (isGTIDMode()) {
            // GTID模式下，CanalLogPositionManager里取最后的gtid，没有则取instanc配置中的
            // 在项目中默认使用的是FailbackLogPositionManager，里面存储了两级缓存
            // 第一级为MemoryLogPositionManager，即现在内存里找，
            // 如果在第一级缓存中找不到，就到第二级缓存，MetaLogPositionManager找
            // 这个也就是我们之前在CanalInstance里设定的metaManager，一般就是从zk中读取
            LogPosition logPosition = getLogPositionManager().getLatestIndexBy(destination);
            if (logPosition != null) {
                return logPosition.getPostion();
            }
			// 如果都找不到，但是用户已经设定了对应的位点信息，则取这个为开始位点
            if (masterPosition != null && StringUtils.isNotEmpty(masterPosition.getGtid())) {
                return masterPosition;
            }
        }
		// 如果都没有找到，则需要去获取mysql中最新的位点了。
        EntryPosition startPosition = findStartPositionInternal(connection);
        if (needTransactionPosition.get()) {
            logger.warn("prepare to find last position : {}", startPosition.toString());
            Long preTransactionStartPosition = findTransactionBeginPosition(connection, startPosition);
            if (!preTransactionStartPosition.equals(startPosition.getPosition())) {
                logger.warn("find new start Transaction Position , old : {} , new : {}",
                    startPosition.getPosition(),
                    preTransactionStartPosition);
                startPosition.setPosition(preTransactionStartPosition);
            }
            needTransactionPosition.compareAndSet(true, false);
        }
        return startPosition;
    }
```
#### 找到相应的位点之后，终于可以开始dump数据了。可以看得代码下面显示定义了一个SinkFunction，这个对象之后会被传入dump的方法中，作为dump过来的数据的处理方法。可以看到dump数据有并发和非并发的区别。先从简单非并发的情况来入手,我们直接分析用Gtid模式来dump的过程，其他的用fileName来dump也是相似的，可以举一反三。
```
	if (isGTIDMode()) {
        // 判断所属instance是否启用GTID模式，是的话调用ErosaConnection中GTID对应方法dump数据
        erosaConnection.dump(MysqlGTIDSet.parse(startPosition.getGtid()), sinkHandler);
    } 
    
    // MysqlConnection.java 
     @Override
    public void dump(GTIDSet gtidSet, SinkFunction func) throws IOException {
    	// 主要是用数据库的连接，先设定一些全局参数的值，例如执行set wait_timeout=9999999
        updateSettings();
        loadBinlogChecksum();
        // 发送位点的gtid信息
        sendBinlogDumpGTID(gtidSet);
		
        DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
        try {
        	// 基于mysql 连接的socket
            fetcher.start(connector.getChannel());
            // 创建解码的组件，传入的参数表示可以解析的LogEvent类型，这时表示接收所有类型的解码
            LogDecoder decoder = new LogDecoder(LogEvent.UNKNOWN_EVENT, LogEvent.ENUM_END_EVENT);
            LogContext context = new LogContext();
            context.setFormatDescription(new FormatDescriptionLogEvent(4, binlogChecksum));
            // fix bug: #890 将gtid传输至context中，供decode使用
            context.setGtidSet(gtidSet);
            // 开始dump数据
            while (fetcher.fetch()) {
                accumulateReceivedBytes(fetcher.limit());
                LogEvent event = null;
                // 解析从socket中读取的数据流，在内部类似nio，也会有一个buffer做缓存
                event = decoder.decode(fetcher, context);
                if (event == null) {
                    throw new CanalParseException("parse failed");
                }
					// 得到event之后，调用之前创建的handler对象，将event放入存储中
                if (!func.sink(event)) {
                    break;
                }
            }
        } finally {
            fetcher.close();
        }
    }
```
#### 在非并发Gtid模式下，sinkFunction中的逻辑如下
```
	public boolean sink(EVENT event) {
        try {
        	// 此处的逻辑最重要的就是调用了        
        	// CanalEntry.Entry event = binlogParser.parse(bod, isSeek);
        	// binlogParser就是我们之前定义的LogEventConvert类，这个里面的逻辑可以后面详述一下，主要就是根据binlog的格式，得到我们想要的结构
            CanalEntry.Entry entry = parseAndProfilingIfNecessary(event, false);

            if (!running) {
                return false;
            }

            if (entry != null) {
                exception = null; // 有正常数据流过，清空exception
                // 在事务buffer中记录一下我们得到的entry
                transactionBuffer.add(entry);
                // 记录一下对应的positions，也就是从entry中得到对应的位点的各种信息
                this.lastPosition = buildLastPosition(entry);
                // 记录一下最后一次有数据的时间
                lastEntryTime = System.currentTimeMillis();
            }
            return running;
        } catch (TableIdNotFoundException e) {
            throw e;
        } catch (Throwable e) {
            if (e.getCause() instanceof TableIdNotFoundException) {
                throw (TableIdNotFoundException) e.getCause();
            }
            // 记录一下，出错的位点信息
            processSinkError(e,
                this.lastPosition,
                startPosition.getJournalName(),
                startPosition.getPosition());
            throw new CanalParseException(e); // 继续抛出异常，让上层统一感知
        }
    }
```
#### 以上就描述了一下建立了一个mysql连接之后，得到binlog数据流，解析成event再解析成我们最终用到的entry的全过程。最后我们可以看到，得到的entry放入了transactionBuffer，貌似没做什么了。那我们再看看这个buffer里面的数据，后面去了哪里。
```
	public void add(CanalEntry.Entry entry) throws InterruptedException {
		// 判断数据的类型，是一个事务的开始，一个事务的结束，或者一个事务中的实际数据
        switch (entry.getEntryType()) {
        	// 若得到了是一个事务的开端，则需要将上一个事务的所有数据处理一下，然后再放入新的事务头
            case TRANSACTIONBEGIN:
                flush();// 刷新上一次的数据
                put(entry);
                break;
          	// 若得到了是一个事务的结束，则需要将事务尾放入buffer，再将当前的事务的所有数据处理一下

            case TRANSACTIONEND:
                put(entry);
                flush();
                break;
            // 如果得到的是事务中的实际数据，则将数据加入buffer
            case ROWDATA:
                put(entry);
                // 针对非DML的数据，直接输出，不进行buffer控制
                // 这里的非DML数据，指的是不是增，删，或者更新的数据
                EventType eventType = entry.getHeader().getEventType();
                if (eventType != null && !isDml(eventType)) {
                    flush();
                }
                break;
            case HEARTBEAT:
                // master过来的heartbeat，说明binlog已经读完了，是idle状态
                put(entry);
                flush();
                break;
            default:
                break;
        }
    }
```
#### 上面一直看到调用了一个叫flush()的方法，那这个方法是做什么的呢？
```
	private void flush() throws InterruptedException {
		// 可以看到用两个AtomicLong类型的值，记录了当前的buffer中已flush到的位置和put的位置
		// 假设此时数组的长度为8，put的位点为7，flush的位点为3
        long start = this.flushSequence.get() + 1;
        long end = this.putSequence.get();
		// 以为上一次flush的位点落后于put的位点，也就是有数据可以flush
        if (start <= end) {
            List<CanalEntry.Entry> transaction = new ArrayList<CanalEntry.Entry>();
            for (long next = start; next <= end; next++) {
            	// 遍历底层数组，通过getIndex方法，得到对应的在数组中的位置
            	// 因为很容易就会遇到指针记录大于了数组的长度，所以需要计算得到真正的位置
                transaction.add(this.entries[getIndex(next)]);
            }
			// 记录了所有需要flush的entry之后，调用回调函数处理，并将flush的指针移到最新位置
            flushCallback.flush(transaction);
            flushSequence.set(end);// flush成功后，更新flush位置
        }
    }
```
#### 简单描述一下上面的过程，就是如果调用了flush方法，就会从底层数组中，得到最新的所有put进来的数据，并用回调函数处理。下面看看这个回调函数是做了什么？
```
	public void flush(List<CanalEntry.Entry> transaction) throws InterruptedException {
        boolean successed = consumeTheEventAndProfilingIfNecessary(transaction);
        // 下面的部分都是做一些记录，不细说了，主要看看上面这句
        if (!running) {
            return;
        }

        if (!successed) {
            throw new CanalParseException("consume failed!");
        }

        LogPosition position = buildLastTransactionPosition(transaction);
        if (position != null) { // 可能position为空
            logPositionManager.persistLogPosition(AbstractEventParser.this.destination, position);
        }
    }
    
    
    protected boolean consumeTheEventAndProfilingIfNecessary(List<CanalEntry.Entry> entrys) throws CanalSinkException,
                                                                                           InterruptedException {
        long startTs = -1;
        boolean enabled = getProfilingEnabled();
        if (enabled) {
            startTs = System.currentTimeMillis();
        }
		// 重点就是这一句了，用eventSink处理flush的所有entry。终于吧eventParser和eventSink有了联系。
        boolean result = eventSink.sink(entrys, (runningInfo == null) ? null : runningInfo.getAddress(), destination);

        if (enabled) {
            this.processingInterval = System.currentTimeMillis() - startTs;
        }

        if (consumedEventCount.incrementAndGet() < 0) {
            consumedEventCount.set(0);
        }

        return result;
    }
```
#### 看看eventSink这个sink方法做了什么

```
    public boolean sink(List<CanalEntry.Entry> entrys, InetSocketAddress remoteAddress, String destination)
                                                                                                throws CanalSinkException,
                                                                                                           InterruptedException {
        return sinkData(entrys, remoteAddress);
    }
    
    
    
    private boolean sinkData(List<CanalEntry.Entry> entrys, InetSocketAddress remoteAddress)
                                                                                            throws InterruptedException {
        // 检查是否有实际有效的数据
        boolean hasRowData = false;
        // 检查是不是心跳数据
        boolean hasHeartBeat = false;
        List<Event> events = new ArrayList<Event>();
        for (CanalEntry.Entry entry : entrys) {
        	// 判断是不是需要直接过滤
            if (!doFilter(entry)) {
                continue;
            }
			// 如果需要过滤事务头尾，则直接过滤，不进行处理
            if (filterTransactionEntry
                && (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND)) {
                long currentTimestamp = entry.getHeader().getExecuteTime();
                // 基于一定的策略控制，放过空的事务头和尾，便于及时更新数据库位点，表明工作正常
                if (lastTransactionCount.incrementAndGet() <= emptyTransctionThresold
                    && Math.abs(currentTimestamp - lastTransactionTimestamp) <= emptyTransactionInterval) {
                    continue;
                } else {
                    lastTransactionCount.set(0L);
                    lastTransactionTimestamp = currentTimestamp;
                }
            }
			// 只要有一个ROWDATA格式的数据，就说明是有实际数据的
            hasRowData |= (entry.getEntryType() == EntryType.ROWDATA);
          // 同理心跳包
            hasHeartBeat |= (entry.getEntryType() == EntryType.HEARTBEAT);
          // 按需求，组装后面需要处理的event
            Event event = new Event(new LogIdentity(remoteAddress, -1L), entry, raw);
            events.add(event);
        }
		
        if (hasRowData || hasHeartBeat) {
            // 存在row记录 或者 存在heartbeat记录，直接跳给后续处理
            return doSink(events);
        } else {
            // 需要过滤的数据
            if (filterEmtryTransactionEntry && !CollectionUtils.isEmpty(events)) {
                long currentTimestamp = events.get(0).getExecuteTime();
                // 基于一定的策略控制，放过空的事务头和尾，便于及时更新数据库位点，表明工作正常
                if (Math.abs(currentTimestamp - lastEmptyTransactionTimestamp) > emptyTransactionInterval
                    || lastEmptyTransactionCount.incrementAndGet() > emptyTransctionThresold) {
                    lastEmptyTransactionCount.set(0L);
                    lastEmptyTransactionTimestamp = currentTimestamp;
                    return doSink(events);
                }
            }

            // 直接返回true，忽略空的事务头和尾
            return true;
        }
    }
```
#### 可以看到，除了过滤掉一些事务的头尾，真正保留下来的数据会组装为event的格式，然后放入一个链表中，统一交到后面的doSink方法处理。
```
	protected boolean doSink(List<Event> events) {
		// 在数据sink前，会有一些自定义的handler来处理
		// 例如在源码中已经添加的HeartBeatEntryEventHandler，专门用于处理心跳包
        for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
            events = handler.before(events);
        }
        long blockingStart = 0L;
        int fullTimes = 0;
        do {
        	// 这边又看到了eventStore，看来整个过程把所有组件都用到了
        	// 具体eventStore中怎么保留这些数据，可以单独开一章说，还是有很多细节的
        	// 这边假设我们已经把events放入对应的底层存储
            if (eventStore.tryPut(events)) {
                if (fullTimes > 0) {
                    eventsSinkBlockingTime.addAndGet(System.nanoTime() - blockingStart);
                }
                for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
                    events = handler.after(events);
                }
                return true;
            } else {
            		// 若失败，则要调用重试策略
                if (fullTimes == 0) {
                    blockingStart = System.nanoTime();
                }
                applyWait(++fullTimes);
                if (fullTimes % 100 == 0) {
                    long nextStart = System.nanoTime();
                    eventsSinkBlockingTime.addAndGet(nextStart - blockingStart);
                    blockingStart = nextStart;
                }
            }

            for (CanalEventDownStreamHandler<List<Event>> handler : getHandlers()) {
                events = handler.retry(events);
            }

        } while (running && !Thread.interrupted());
        return false;
    }
```
#### 综上，我们可以总结一下整个拉取数据的过程做了什么操作
- 利用eventParse组件，先建立对应的mysql的socket连接
- 利用两级缓存，(1:内存中的位点缓存  2: 在zk中的位点缓存 3: 用户自定义的位点信息) 依次往下寻找，找到开始拉取数据的位点信息
- 以GTID模式为例，当得到了gtid位点后，用mysql的socket连接，得到数据并进行decode操作为相应的event
- 将event加入transactionBuffer，并在合适的时刻将数据全都flush
- flush的过程就是eventSink组件发挥作用的时候
- sink的过程即为按需求过滤到事务的头和尾，已经判断是否为心跳包
- 然后把所有需要的event，先用CanalEventDownStreamHandler来预处理一遍
- 然后用eventStore来存储我们的events

#### 这个eventStore就是我们的event最后出口的地方，可以由canalServer搭建netty服务端，由canalClient来发送拉取event的请求，或者像在otter中的做法。把eventStore中的所有数据，都作为数据源，再进行后面的extract，transform，load作用，输出到我们想要的地方。
#### 好了，简单过了一遍整个的拉取过程，留了几个问题
- decode从mysql拉取得到的binlog信息的具体过程
- eventStore底层怎么存储events

#### 以上两个问题会在后面的文章讲到