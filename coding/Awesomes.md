## 分布式系统架构
### 论文
[CAP](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)
[BASE](http://queue.acm.org/detail.cfm?id=1394128)
[Gossip](https://en.wikipedia.org/wiki/Gossip_protocol)

### 分布式事务

## 微服务架构

### 微服务
[microservices](http://microservices.io/patterns/microservices.html)
[microservices by Martin Folwer](https://martinfowler.com/articles/microservices.html)

### 服务注册与发现
Service Discovery:
- [Client Side Discovery](http://microservices.io/patterns/client-side-discovery.html)
- [Server Side Discovery](http://microservices.io/patterns/server-side-discovery.html)

Service registry:
- [Self-Registration Pattern](http://microservices.io/patterns/self-registration.html)
- [Third-Party Registration Pattern](http://microservices.io/patterns/3rd-party-registration.html)

[[服务注册发现与调度](https://segmentfault.com/a/1190000006175561)](https://segmentfault.com/a/1190000006175561)

Resources: 
[Service Discovery for NGINX Plus Using Consul APIs](https://www.nginx.com/blog/service-discovery-with-nginx-plus-and-consul/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
[Service Discovery for NGINX Plus with etcd](https://www.nginx.com/blog/service-discovery-nginx-plus-etcd/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
[Service Discovery for NGINX Plus with ZooKeeper](https://www.nginx.com/blog/service-discovery-nginx-plus-zookeeper/?utm_source=service-discovery-nginx-plus-srv-records-consul-dns&utm_medium=blog&utm_campaign=DevOps)
[Service Discovery for NGINX Plus Using DNS SRV Records from Consul](https://www.nginx.com/blog/service-discovery-nginx-plus-srv-records-consul-dns/)

### 工具
服务注册与发现：
- [Euraka](https://github.com/Netflix/eureka)
- [Consul](https://github.com/consul/consul)
- [Etcd](https://github.com/coreos/etcd)
- [ZooKeeper]()

服务降级熔断
- [Hystrix](https://github.com/Netflix/Hystrix)

服务调度
- [Ribbon](https://github.com/Netflix/ribbon)


## 消息系统
### 消息协议
[MsgPack](https://github.com/msgpack/msgpack/blob/master/spec.md)
[ProtoBuffer](https://developers.google.com/protocol-buffers/docs/proto3)
[FlatBuffer](https://google.github.io/flatbuffers/)
[kryo](https://github.com/EsotericSoftware/kryo)
[avro](http://avro.apache.org/docs/current/)
[json-dsl-platform](https://dsl-platform.com/)

XMPP
MTProto

### Kafka
[Kafka剖析（一）：Kafka背景及架构介绍](http://www.infoq.com/cn/articles/kafka-analysis-part-1?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
[Kafka设计解析（二）：Kafka High Availability （上）](http://www.infoq.com/cn/articles/kafka-analysis-part-2?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
[Kafka设计解析（三）：Kafka High Availability （下）](http://www.infoq.com/cn/articles/kafka-analysis-part-3?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
[Kafka设计解析（四）：Kafka Consumer解析](http://www.infoq.com/cn/articles/kafka-analysis-part-4?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
[Kafka设计解析（五）：Kafka Benchmark](http://www.infoq.com/cn/articles/kafka-analysis-part-5?utm_source=infoq&utm_campaign=user_page&utm_medium=link)

[The Log：每个程序员都应该知道有关实时数据的统一抽象（1）概念](http://blog.jobbole.com/89674/)
[The Log：每个程序员都应该知道有关实时数据的统一抽象（2） 数据集成](http://blog.jobbole.com/89688/)
[The Log：每个程序员都应该知道有关实时数据的统一抽象（3）日志与实时流处理](http://blog.jobbole.com/89703/)
[The Log：每个程序员都应该知道有关实时数据的统一抽象（4）系统构建](http://blog.jobbole.com/89711/)

### RabbitMQ

### ZeroMQ

## 网络模型
ZeroMQ 组网模型

## 架构模型
### 六边形架构 

### Event Sourcing


C10K

## 压测工具
[wrk](https://github.com/wg/wrk/wiki/Installing-Wrk-on-Linux)
[Web服务器性能/压力测试工具http_load、webbench、ab、Siege使用教程](https://www.vpser.net/opt/webserver-test.html)


[Functional Reactive Programming](https://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming)

[web框架性能比拼](https://www.techempower.com/benchmarks/#section=intro&hw=ph&test=fortune)

## Reactive Programming
https://github.com/reactive-streams/reactive-streams-jvm
https://github.com/ReactiveX/RxJava
akka stream
http://www.reactivemanifesto.org/
http://www.reactive-streams.org/


cassandra:
http://www.ebaytechblog.com/2012/07/16/cassandra-data-modeling-best-practices-part-1/
http://www.ebaytechblog.com/2012/08/14/cassandra-data-modeling-best-practices-part-2/
中译：
http://www.infoq.com/cn/articles/best-practice-of-cassandra-data-model-design
http://www.infoq.com/cn/articles/best-practices-cassandra-data-model-design-part2

[亿级流量网站架构核心技术](http://jinnianshilongnian.iteye.com/blog/2347183)




## 面试
[面试前需要做哪些准备？] (http://daily.zhihu.com/story/9365229?utm_campaign=in_app_share&utm_medium=Android&utm_source=com.baiji.jianshu.ui.editor.EditorActivityV19)

Reactive Programming
rxjava

## Pattern
[Observer pattern](https://en.wikipedia.org/wiki/Observer_pattern)