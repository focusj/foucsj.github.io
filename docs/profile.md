## 个人信息

- 王康 / 男 / 5年 工作经验 / 本科
- 电话：18610842638； Email：focusj.x@gmail.com
- 博客：http://www.jianshu.com/users/13542edebea3/

## 工作经历

2016年03月~至今         北京第三石信息科技有限公司  后端工程师

2014年10月~2015年11月   ThoughtWorks            全栈工程师

2012年04月~2014年09月   北京尤尼信息科技有限公司    全栈工程师

## 项目经验

### 后端服务拆分

*职责*: 负责人

*技术选型*: Java8, Vert.x, JOOQ, MySQL, Redis, Docker

*项目背景*

北京第三石是成立了三年多的创业公司, 由于缺乏对后端服务重构和优化, 导致系统臃肿复杂, 到处是垃圾代码. 开发维护成本越来越高. 而且Python + Django也导致了严重的性能问题.为了解决这些问题, 开始推进后端服务的拆分.

*项目经历*

- 通过对Spring Boot和Vert.x进行性能测试, 以及对Vert.x风险的评估, 最终敲定Vert.x作为构建服务的基础框架.

- 使用Redis实现数据缓存, 实现时避免了缓存穿透, 缓存并发问题.

- 提供符合RESTful规范的API.

现在项目已经运行在测试环境, 准备上线.

### 开放API服务

*职责*: 负责人

*技术选型*: Java8, Vert.x, JOOQ, MySQL, Redis

*项目背景*

该项目是为合作大商家提供的开放API服务, 这些服务商通常需要维护很多的用户和商品. 通过开放API可以实现高效的用户和商品管理.

*项目经历*：

- 搭配使用AWS的API Gateway和API Key实现用户的鉴权与访问频次限制.

- 引入Vert.x Circuit Breaker负责和内部用户商品服务交互.

该项目已经上线

### 订单系统

*职责*: 负责人

*技术选型*: Java 8, Spring Boot, Spring Statemachine, JOOQ, Akka, Flyway, RabbitMQ, Redis, MySQL

*项目背景*

5miles主要在美国市场做C2C业务, 主要是线下二手物品交易. 为了让用户交易更安全方便, 公司决定要做担保交易和物流. 由于引入了Java语言, 所以订单系统要作成独立新的系统.

*项目经历*：

- 提供符合RESTful规范的API.

- 引入Akka实现异步任务处理, 下单接口从原来的3秒提升到了1秒左右.

- 基于Redis实现分布式锁.

- 分布式事务是如何操作的.

- 引入Spring Statemachine控制订单状态流转.

项目去年已经上线.

### 名称：业务系统

*职责*：全栈开发

*技术选型*：.Net Web API, ReactJS, SQL Server

*项目背景*

该项目是ThoughtWorks为某美国公司开发的业务处理系统. 该公司主要负责为第三方公司提供财务审计, 国际报税等业务. 每项业务都由单独的系统处理. 为了方便第三方客户查看及使用, 需要作一个聚合的网站, 把现有的业务聚合到一个网站内操作.

*项目经历*：

- 由于依赖了众多其它服务, 引入Pact作系统间集成测试.

- 用ReactJS解决前端控件复用的问题.

- 使用Redis做Session管理.

## 技术总结

- 熟悉Java, Scala语言.

- 了解多线程编程, 异步编程.

- 了解JVM及常见垃圾回收算法.

- 缓存, RESTful API, 微服务, 分布式系统

- 框架和工具： Spring Boot, Vert.x, Netty, Akka, Redis, RabbitMQ, Play FrameWork, MongoDB.

## 工作期望

Java/Scala工程师; 后端研发工程师;