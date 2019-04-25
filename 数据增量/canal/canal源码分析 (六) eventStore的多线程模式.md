# canal源码分析 (六) eventStore的多线程模式
## 前言
#### 上一章说了一下单线程模式，这章会继续说一下多线程模式。因为多线程模式主要用到了Disruptor，所以本文也会详细说一下Disruptor的内容

## 正文
### 多线程模式
#### 回到AbstractEventParser中
```
	if (parallel) {
        // build stage processor
        multiStageCoprocessor = buildMultiStageCoprocessor();
        ...
    }
    
   
    // AbstractMysqlEventParser.java
    protected MultiStageCoprocessor buildMultiStageCoprocessor() {
        MysqlMultiStageCoprocessor mysqlMultiStageCoprocessor = new MysqlMultiStageCoprocessor(parallelBufferSize,
            parallelThreadSize,
            (LogEventConvert) binlogParser,
            transactionBuffer,
            destination);
        mysqlMultiStageCoprocessor.setEventsPublishBlockingTime(eventsPublishBlockingTime);
        return mysqlMultiStageCoprocessor;
    }
```
#### 可以大概看到，多线程模式下会先构建一个MultiStageCoprocessor，这个可以类比为单线程情况下的SinkHandler。直接看到GTID模式下的dump方法
```
	multiStageCoprocessor.start();
    erosaConnection.dump(gtidSet, multiStageCoprocessor);
```
#### 这次我们先看到dump方法
```
	@Override
    public void dump(GTIDSet gtidSet, MultiStageCoprocessor coprocessor) throws IOException {
        updateSettings();
        loadBinlogChecksum();
        sendBinlogDumpGTID(gtidSet);
        ((MysqlMultiStageCoprocessor) coprocessor).setConnection(this);
        ((MysqlMultiStageCoprocessor) coprocessor).setBinlogChecksum(binlogChecksum);
        DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
        try {
            fetcher.start(connector.getChannel());
            while (fetcher.fetch()) {
                accumulateReceivedBytes(fetcher.limit());
                LogBuffer buffer = fetcher.duplicate();
                fetcher.consume(fetcher.limit());
                if (!coprocessor.publish(buffer)) {
                    break;
                }
            }
        } finally {
            fetcher.close();
        }
    }
```
#### 整体逻辑和单线程模式下都是类似的。重点看到
```
	coprocessor.publish(buffer)
```
#### 好了，现在专心回到MultiStageCoprocessor类中。我们先要看到start方法，再看到publish方法。