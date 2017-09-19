# H

## H2

ThreadLocal是一个工具类，用于设置线程本地缓存。ThreadLocal提供了两个方法：set和get，分别用户设置和获取缓存。

在Thread内部定义了`ThreadLocal.ThreadLocalMap threadLocals = null;`用于存储线程私有的数据。

我们把Thread理解为纵向的，则Thread则可以理解为横向的，贯穿多个Thread。它们的交点可以定位一个数据。