akka集群有两种启动方式。一种是手动加入节点(在akka中节点叫做Node); 另一种是通过在配置中指定seed node。seed node是集群的通信节点，用来进行集群的创建和选举。通常我们会在配置文件中配置一系列的seed node，当新的节点想要加入集群时，只要与其中任何一个取得通信即可。需要注意，当启动第一个节点时，这个节点一定要配置在seed node第一个位置。

在继续下文之前先介绍一下我们目前的服务情况。我们有三台WEB Server，这三台server挂在Amazon的ELB上。这三台server都是无状态的。项目部署使用code deploy，由于三台server都是无状态的，所以部署非常简单。从ELB任意摘下一台server，重启，然后再挂到ELB。

当我们尝试引入akka cluster到项目中时，现有的无状态部署方式便不再适用了。首先我们最初的期望是引入集群但尽量少带来项目启动的复杂性。手动加入节点的方式被淘汰。而使用seed node现有部署脚本就要改。因为seed node决定了一定要有一台服务器优先启动。

纠结了一段时间后，想到一个解决方案。借助redis来启动集群。redis可以做两件事情，分布式锁和存储seed node。分布式锁的意义在于防止多台server同时启动造成cluster分裂。这个方案具体细节是:当节点启动时，首先问redis是否已有启动节点，如果没有则以自己为seed node创建集群;如果已有seed，则加入集群。这看起来是一个可行的方案。在写第一行代码之前，本着不重复造轮子的原则，去Google groups search了一下，找到了constructR这个工具。

当我看了它的介绍之后发现和我们方案的核心思路是一样的，但是constructR实现的更精细一些。它抽象出一个状态机用于控制集群启动是状态的流转。我直接盗图了：

![Notes_1483621897775.png](http://upload-images.jianshu.io/upload_images/78847-9a106c3d4f0b8fb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在状态流转的过程中，任何一步出现异常都会停掉当前actor system。ConstructR官方给出的底层存储是etcd，但是也有consul，redis，和zookeeper的实现。

ConstructR的使用也非常简单：第一步添加依赖，第二步添加ConstructR akka的扩展，第三步配置ConstructR。官方文档已经提供了非常完整的基于etcd和sbt的配置，下边我列出我基于redis和gradle的配置：

```
// dependency
repositories {
    jcenter()
    mavenCentral()
    maven { url "https://dl.bintray.com/everpeace/maven/" }
}
dependencies {
    compile('de.heikoseeberger:constructr-akka_2.11:0.13.2')
    compile('com.github.everpeace:constructr-coordination-redis_2.11:0.0.1')
}

// add constructr extension for akka 
akka.extensions = [ "de.heikoseeberger.constructr.akka.ConstructrExtension" ]

// constructr conf
redis {
  host = "localhost"
  port = 6379
  db = 5
}

constructr {
  coordination {
    class-name = "com.github.everpeace.constructr.coordination.redis.RedisCoordination"
    host = ${redis.host}
    port = ${redis.port}
    redis {
      db = ${redis.db}
    }
  }

  coordination-timeout = 3 seconds  // Maximum response time for coordination service (e.g. etcd)
  join-timeout         = 15 seconds // Might depend on cluster size and network properties
  max-nr-of-seed-nodes = 0          // Any nonpositive value means Int.MaxValue
  nr-of-retries        = 2          // Nr. of tries are nr. of retries + 1
  refresh-interval     = 30 seconds // TTL is refresh-interval * ttl-factor
  retry-delay          = 3 seconds  // Give coordination service (e.g. etcd) some delay before retrying
  ttl-factor           = 2.0        // Must be greater or equal 1 + ((coordination-timeout * (1 + nr-of-retries) + retry-delay * nr-of-retries)/ refresh-interval)!
}
```

---
write on 2017-1-5