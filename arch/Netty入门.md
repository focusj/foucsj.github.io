# Netty

## Netty中的EventLoop

EventLoop是Netty处理并发的底层抽象。顾名思义，EventLoop的工作原理是不断的loop event。

- EventLoopGroup包含多个EventLoop

- EventLoop创建的时候会绑定唯一的一个Thread

- 所有EventLoop要处理的Events都由其绑定的Thread处理

- 一个Channel只和一个EventLoop绑定，一个EventLoop可以处理一到多个Channel

![netty-componets](images/Selection_041.png)