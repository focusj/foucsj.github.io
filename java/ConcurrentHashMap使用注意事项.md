CHM是线程安全的数据结构，即可以用在多线程环境下。但是在写入的时候有一点需要注意一下，否则仍然会产生[race condition](https://en.wikipedia.org/wiki/Race_condition)（竞态条件）。

先说一下我的使用场景：CHM缓存一分钟内的监控指标数据，当插入新的指标数据时，先查看缓存中是否已经有改指标项，如果有则新老数据聚合。

因为已经习惯了先检查再插入的写法，直觉上代码就写成了下边的样子：

```java
private MetricHolder getOrCreate() {
    MetricHolder e = metricBuffer.getOrDefault(newHolder.key(), null);
    if(!map.ContainsKey(newHolder.key()))
        return metricBuffer.put(newHolder.key(), newHolder);
    else 
        return map.get(key);
}
```

但是这种实现在多线程环境下可能会引发错误。想以下场景：两个线程同时进入了getOrCreate方法，都判断当前CHM中不存在要插入的数据，那两个线程都会执行put的逻辑，两个线程一定有一个覆盖了另一个刚写入的数据。

正确的实现，看CHM的源码，putVal方法有一个onlyIfAbsent的参数。

```java 
final V putVal(K key, V value, boolean onlyIfAbsent) {
```

利用这个参数我们能控制只有当前key为空的情况下才执行put动作，否则返回当前key对应的value。好吧，这会儿应该可以想起JDK8提供了putIfAbsent这个方法。汗。。。所以这个方法等价于：

```java
 if (!map.containsKey(key))
    return map.put(key, value);
 else
    return map.get(key);
 }
```

改进实现方案：

```java
MetricHolder createOrGet = metricBuffer.putIfAbsent(newHolder.key(), newHolder);
if (createOrGet == null) {
    return newHolder;
} else {
    if (!builder.isInstance(createOrGet.getMetric())) {
        throw new IllegalArgumentException(name + " is used for another different type metric.");
    }
    return createOrGet;
}
```

