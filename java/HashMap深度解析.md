### HashMap的数据结构

HashMap是数组和链表的组合结构，数组用于存储哈希值，每个哈希值代表了一个bucket。bucket是链表结构，之所以是链表结构是效率和空间权衡的结果。



说两个键值对哈希碰撞，那他们会进入同一个bucket。那么，再索引这个key的时候就会便利整个链表，因此哈希碰撞越严重，整个链表的读取效率越低。



### HashMap的构建需要注意的事情

- 初始容量(initial size)。HashMap构建的时候可以传入初始容量，初始容量必须是2的幂。比如设置初始容量为5, 则经过处理之后初始容量则会变成8

具体的算法是这样的：

```

static final int tableSizeFor(int cap) {

    int n = cap - 1;

    n |= n >>> 1;

    n |= n >>> 2;

    n |= n >>> 4;

    n |= n >>> 8;

    n |= n >>> 16;

    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;

}

```



- 加载因子(loading factor)。loading factor决定HashMap什么时候进行扩容。我们每次put元素的时候，会进行检查如果当前元素的个数，如果当前的元素个数 > 容量 × loading factor，那么HashMap会进行扩容，新的容量是当前容量的2倍。扩容是比较耗时的操作，因此如果使用HashMap存储较多的元素，应事先估算好HashMap的容量。



- 最大容量。1 << 30 = pow(2, 30)即2的30次幂。



### HashMap不能用在并发环境下

HashMap不能用在并发环境下，并发环境应使用ConcurrentHashMap。并发环境下使用HashMap会造成cpu 100的问题，病灶在HashMap扩容的时候，可能产生环形链表，因此当有县城访问这个key的时候就产生了Infinite Loop，CPU飙100%。[详细解释](http://coolshell.cn/articles/9606.html)。