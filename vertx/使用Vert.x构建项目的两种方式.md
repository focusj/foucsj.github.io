# 使用Vertx 构建项目的两种方式

## One Whole Stack

这种方式是把所有的逻辑放到一个Verticle中，类似于我们传统的方式。

这种方式，如果我们构建大项目的话比会用到类似DI这种工具。

## Service As Verticle

把项目抽象成小的Service，类似与六边形架构，每个依赖或者模块都是服务（或者叫Adaptor）。各个服务之间通过EventBus进行通信。这是一种有想象力的架构。