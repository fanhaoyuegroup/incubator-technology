# Canal简介：
cannal是阿里纯Java开发，基于数据库增量日志解析，提供增量数据订阅&消费，目前主要支持了mysql

## Canal原理 
canal模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议
mysql master收到dump请求，开始推送binary log给slave(也就是canal)
canal解析binary log对象(原始为byte流)
![image](http://incdn1.b0.upaiyun.com/2017/06/bbbebcd656dbd2f628ec8d7b27450606.png)
## Canal说明
 1：canal的原理是基于mysql binlog技术，所以这里一定需要开启mysql的binlog写入功能，建议配置binlog模式为row.
 binlog_format 必须设置为 ROW, 因为在 STATEMENT 或 MIXED 模式下, Binlog 只会记录和传输 SQL 语句（以减少日志大小），而不包含具体数据，我们也就无法保存了。
 mysql的binlog是多文件存储，定位一个LogEvent需要通过binlog filename + binlog position，进行定位
 mysql的binlog数据格式，按照生成的方式，主要分为：statement-based、row-based、mixed。
 
 mysql> show variables like 'binlog_format';
 
 +---------------+-------+  </br>
 | Variable_name | Value | </br>
 +---------------+-------+ </br>
 | binlog_format | ROW   | </br>
 +---------------+-------+ </br>
 1 row in set (0.00 sec) </br>
 
 目前canal支持所有模式的增量订阅(但配合同步时，因为statement只有sql，没有数据，无法获取原始的变更日志，所以一般建议为ROW模式)
 2：canal的原理是模拟自己为mysql slave，所以这里一定需要做为mysql slave的相关权限.

==CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, SHOW VIEW, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
-- GRANT ALL PRIVILEGES ON *.* TO 'canal'@'%' ; 需要具有SHOW VIEW 权限
FLUSH PRIVILEGES;==
## mysql主从复制说明
**复制过程：** 
主节点必须启用二进制日志，记录任何修改数据库数据的事件。
从节点开启一个线程（I/O Thread)把自己扮演成mysql的客户端，通过mysql协议，请求主节点的二进制日志文件中的事件
主节点启动一个线程（dump Thread），检查自己二进制日志中的事件，跟对方请求的位置对比，如果不带请求位置参数，则主节点就会从第一个日志文件中的第一个事件一个一个发送给从节点。
从节点接收到主节点发送过来的数据把它放置到中继日志（Relay log）文件中。并记录该次请求到主节点的具哪个二进制日志文件的哪个位置。
从节点启动另外一个线程（sql Thread ），把replaylog中的事件读取出来，并在本地再执行一次。
复制中线程的作用：
**从节点：**

I/O Thread:从Master请求二进制日志事件，并保存于中继日志中。
Sql Thread:从中继日志中读取日志事件，在本地完成重放。
**主节点：**

Dump Thread:为每个Slave的I/O Thread启动一个dump线程，用于向从节点发送二进制事件。
