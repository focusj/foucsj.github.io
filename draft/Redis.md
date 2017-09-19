# Redis

## 基础命令：
    
    内部数据结构
    使用场景

### String

### Hash

### List

### Set

### ZSet

### BitMap

### Geo

Geo底层使用set存储。

### HyperLogLog

基于基数计数法，空间利用极小，存在错误率。

## 高级用法

### Redis Pipeline

pipeline可以批量的执行多个redis命令，比如客户端批量的向server set key。记住pipeline不保证原子性。

pipeline也可以支持导入文件命令，注意Redis只支持dos的`\r\n`。unix2dos可以转换。

[example](https://github.com/focusj/tutorials/blob/master/src/main/java/redis/PipeExample.java)

### Lua

### Pub/Sub

使用Redis可以实现简单的订阅功能。它只能实现在线消息的发布，不支持订阅管理，消息历史，ack等等，使用时要先分析自己的场景。

订阅主题：subscribe CHANNEL_NAME。subscribe可以支持简单的模式匹配功能来订阅一组消息：psubscribe PREFIX*。

unsubscribe和punsubscribe可以取消订阅主题。

[example](https://github.com/focusj/tutorials/blob/master/src/main/java/redis/PubSubExample.java)

### 分布式事物

单节点模式，多节点模式。

## 持久化

### AOF

触发时机：手动执行bgsave，命令达到速度阀值。

### RDB

## 部署

### 主从

### 哨兵

### 集群

## 运维