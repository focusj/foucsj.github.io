# Reactor 模型

![Reactor-Model](../images/Reactor.png)

- Resource：资源指的是提供系统输入或者消费系统输出的资源。比如Socket套接字。
- Demultiplexer：事件分离器负责对资源进行轮寻等待，当资源ready的时候，分离器负责将资源发送给Dispatcher。
- Dispatcher：处理Handler的注册和反注册。当资源到达时负责把资源分发到相应的Handler中。
- Handler：对资源处理逻辑的封装。

Reactor的优势是：分离IO操作和程序逻辑；用少量的线程处理大量的IO请求。

坏处是：基于事件编程debug比较费力；编程难度变大。