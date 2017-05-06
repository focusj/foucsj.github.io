[上篇文章-使用akka作异步任务处理](http://www.jianshu.com/p/b32e60d209a5)主要讲了如何使用Akka作异步任务处理。最后还抛出一个问题。

具体问题的描述就不在这篇文章赘述了，我们仅简单回顾一下第一种解决方案：覆写persistenceId()时，加一个UUID，这样三台服务器上的Actor就不会再共享journal。虽然这个方案已经可以解决问题了，但并不是最理想的。首先，现在的项目中只是用akka处理一些无状态的任务异步处理，但是将来肯定要用akka作更多的事情。比如，缓存，DAO这些可以设计成有状态的，而现在actor重启时是不能replay消息消息历史的，所以这样不能最大限度发挥actor的优势。还有，我的目标是把所有的后端server构建为一个逻辑上的server，现在他们仍然是三个各自为营的独立server。因此我又继续作了一些调研，最终发现了Cluster Singleton。

文档上给出了Cluster Singleton的适用场景：
- 集群中的单点决策，或者协调
- 统一外部系统出口
- 一主多从
- 统一命名服务或路由逻辑

第二点正好就是我们的场景。下边看一下如何使用Cluster Singleton。

1. 添加依赖（我用的构建工具是Gradle）

```
compile("com.typesafe.akka:akka-cluster_2.11:${akkaVersion}")
compile("com.typesafe.akka:akka-cluster-tools_2.11:${akkaVersion}")
```

2. 创建actor

```
actorSystem.actorOf(ClusterSingletonManager.props(
    SpringExtProvider.get(actorSystem).props("NotificationActor"),
    PoisonPill.getInstance(),
    ClusterSingletonManagerSettings.create(actorSystem).withRole("master")
), "notification-master");

notificationActor = actorSystem.actorOf(
    ClusterSingletonProxy.props(
        "/user/notification-master",
        ClusterSingletonProxySettings.create(actorSystem).withRole("master")),
    "notification-proxy");
```
创建actor比原来复杂了。首先要创建一个ClusterSingletomManager。ClusterSingletonManager也是一个Actor，它会在Cluster的每个节点上都启动起来（或者集群拥有某些角色的节点）。ClusterSingletomManager.props要传入三个参数，第一个是需要创建Singleton实例的Actor配置；第二个是当Manager关闭时要给它管理的Actor发送什么消息；第三个，集群部署配置，即指定ClusterSingletomManager在哪些集群Node上启动。

接下来ClusterSingletonManager会选择一个最老的实例并在上面创建Actor单例。ClusterSingletonManager可以确保整个集群中至多有一个singleton的实例，言下之意，存在没有singleton实例的时刻。比如cluster node crash，原有的singleton实例丢失，这时需要重新选举新最老的ClusterSingletonManager，然后创建新的singleton实例。

访问Singleton Actor需要借助于ClusterSingletonProxy，ClusterSingletonProxy会把所有的消息forward给当前被代理的Actor实例。因为有可能某些时刻是没有singleton actor实例的，所以遇到这种情况ClusterSingletonProxy会先把消息缓存，当新的Actor单例创建之后，再把缓存的消息转发过去。

当然，Cluster Singleton也有它的问题，比如单点潜在的性能问题，而且singleton actor并不是100%可用。但相比于第一种方案，显然这个要更接近我的期望。

---
write on 2017-1-10