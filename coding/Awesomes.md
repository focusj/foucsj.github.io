# 常用资料索引

## 分布式系统架构

### 论文

日志：构建分布式系统基石。

- [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [CAP](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)
- [BASE](http://queue.acm.org/detail.cfm?id=1394128)
- [Gossip](https://en.wikipedia.org/wiki/Gossip_protocol)

### 事务 & 锁

- [基于Redis的分布式锁到底安全吗（上）？](https://mp.weixin.qq.com/s/JTsJCDuasgIJ0j95K8Ay8w)
- [基于Redis的分布式锁到底安全吗（下）？](https://mp.weixin.qq.com/s/4CUe7OpM6y1kQRK8TOC_qQ)


## 微服务架构

### 微服务

- [microservices](http://microservices.io/patterns/microservices.html)
- [microservices by Martin Folwer](https://martinfowler.com/articles/microservices.html)

### 服务注册与发现

理论与模式：

- [服务注册发现与调度](https://segmentfault.com/a/1190000006175561) 
- [Client Side Discovery](http://microservices.io/patterns/client-side-discovery.html)
- [Server Side Discovery](http://microservices.io/patterns/server-side-discovery.html)
- [Self-Registration Pattern](http://microservices.io/patterns/self-registration.html)
- [Third-Party Registration Pattern](http://microservices.io/patterns/3rd-party-registration.html)
- [Service Discovery for NGINX Plus Using Consul APIs](https://www.nginx.com/blog/service-discovery-with-nginx-plus-and-consul/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
- [Service Discovery for NGINX Plus with etcd](https://www.nginx.com/blog/service-discovery-nginx-plus-etcd/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
- [Service Discovery for NGINX Plus with ZooKeeper](https://www.nginx.com/blog/service-discovery-nginx-plus-zookeeper/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
- [Service Discovery for NGINX Plus Using DNS SRV Records from Consul](https://www.nginx.com/blog/service-discovery-nginx-plus-srv-records-consul-dns/)

工具及解决方案：

- [Euraka](https://github.com/Netflix/eureka)
- [Consul](https://github.com/consul/consul)
- [Etcd](https://github.com/coreos/etcd)
- [ZooKeeper](https://zookeeper.apache.org/)

### 服务调用

服务调用框架，内含熔断降级逻辑。Hystrix已是成熟的方案，Vert.x中也有一套简单实现。

- [Hystrix](https://github.com/Netflix/Hystrix)

## 消息系统

### 消息协议

- [MsgPack](https://github.com/msgpack/msgpack/blob/master/spec.md)
- [ProtoBuffer](https://developers.google.com/protocol-buffers/docs/proto3)
- [FlatBuffer](https://google.github.io/flatbuffers/)
- [kryo](https://github.com/EsotericSoftware/kryo)
- [avro](http://avro.apache.org/docs/current/)
- [json-dsl-platform](https://dsl-platform.com/)
- [BSON](http://bsonspec.org/)
- XMPP(用于会话聊天)
- MTProto(用于会话聊天)

### Kafka

- [Kafka剖析（一）：Kafka背景及架构介绍](http://www.infoq.com/cn/articles/kafka-analysis-part-1?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
- [Kafka设计解析（二）：Kafka High Availability （上）](http://www.infoq.com/cn/articles/kafka-analysis-part-2?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
- [Kafka设计解析（三）：Kafka High Availability （下）](http://www.infoq.com/cn/articles/kafka-analysis-part-3?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
- [Kafka设计解析（四）：Kafka Consumer解析](http://www.infoq.com/cn/articles/kafka-analysis-part-4?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
- [Kafka设计解析（五）：Kafka Benchmark](http://www.infoq.com/cn/articles/kafka-analysis-part-5?utm_source=infoq&utm_campaign=user_page&utm_medium=link)

Kafka 作者描述其出现背景，解决的核心问题以及核心理念。

- [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [The Log：每个程序员都应该知道有关实时数据的统一抽象（1）概念](http://blog.jobbole.com/89674/)
- [The Log：每个程序员都应该知道有关实时数据的统一抽象（2） 数据集成](http://blog.jobbole.com/89688/)
- [The Log：每个程序员都应该知道有关实时数据的统一抽象（3）日志与实时流处理](http://blog.jobbole.com/89703/)
- [The Log：每个程序员都应该知道有关实时数据的统一抽象（4）系统构建](http://blog.jobbole.com/89711/)

### RabbitMQ

- [RabbitMQ Documentation](http://rabbitmq.mr-ping.com/)

### 压测工具

ab简单常用，Siege可以随机从一些URLs随机选择进行测试，可能会更偏向实际系统吞吐。

- [wrk](https://github.com/wg/wrk/wiki/Installing-Wrk-on-Linux)
- [Web服务器性能/压力测试工具http_load、webbench、ab、Siege使用教程](https://www.vpser.net/opt/webserver-test.html)

## 程序 & 范式

### Reactive Programming

- [Reactive Streams](http://www.reactive-streams.org/)
- [Reactive Manifesto](http://www.reactivemanifesto.org/)
- [RxJava](https://github.com/ReactiveX/RxJava)
- [Observer Pattern](https://en.wikipedia.org/wiki/Observer_pattern)
- [Iterator Pattern](https://en.wikipedia.org/wiki/Iterator_pattern)
- [Reactive Extensions](https://en.wikipedia.org/wiki/Reactive_extensions)
- [Functional Reactive Programming](http://conal.net/talks/)

### Coding

《代码整洁之道》，《重构》

## 存储

### Cassandra

Cassandra数据模型设计

- [cassandra data modeling best practices-part 1](http://www.ebaytechblog.com/2012/07/16/cassandra-data-modeling-best-practices-part-1/)
- [cassandra data modeling best practices-part 2](http://www.ebaytechblog.com/2012/08/14/cassandra-data-modeling-best-practices-part-2/)

中译：

- [第一部分](http://www.infoq.com/cn/articles/best-practice-of-cassandra-data-model-design)
- [第二部分](http://www.infoq.com/cn/articles/best-practices-cassandra-data-model-design-part2)

### Redis

《Redis开发与运维》案头著作