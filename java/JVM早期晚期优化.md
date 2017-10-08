## Java 范型

Java范型只是一个语法糖, 在编译之后类型参数都变成原始类型. 这称之为类型擦除. 因此List<Integer>和List<String>在编译后是同一个类型. 

## 自动装箱和自动拆箱

java中分为基础类型和包装类型, 包装类型在运行是都会进行自动拆包. 并且会对Integer进行缓存, 范围是-127 ~ 127. 这个范围可以进行调整. 具体实现类是IntegerCache. 除了Integer, Long, Short, Byte都有缓存.

## JIT
解释器可以立即执行代码省去编译的时间, 让程序快速的执行；随着程序的运行编译器可以把更多的代码编译成本地代码提升运行速度. 我们可以通过JVM参数设置开启那种模式, 默认是混合模式. 
```sh
 ~/3stone/ico-api   master ●✚  java -Xcomp -version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, compiled mode)
 ~/3stone/ico-api   master ●✚  java -Xint -version 
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, interpreted mode)
 ~/3stone/ico-api   master ●✚  java -version      
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)

```

分层编译(Tiered Compilation): 
0层: 程序解释执行, 解释器不开启性能监控功能, 可触发第一层编译. 
1层: 将字节码编译成本地代码, 运行简单, 可靠的优化, 如果有必要则开启性能监控.
2层: 将自己码编译成本地代码, 运行耗时较长的优化.

编译对象和出发条件: 被多次调用的方法；被多次执行的循环体. 每个方法都会建立一个计数器, 用于记录该方法是否达到了编译的阀值. 如果达到则将该方法编译成本地代码.-XX:CompileThreshold可以设置阀值大小.

如果不做任何设置, 方法调用计数器统计的并不是方法调用的绝对次数, 而是一个相对的执行频率. 当超过一定的时间如果该方法仍没达到编译的限制, 则统计计数器减少一半. 称为热度衰减.

回边计数器它的作用是统计一个方法中循环体代码执行的次数. 回边计数器没有热度衰减.

编译优化技术:
- 方法内联. 它的好处是减少方法调用的成本, 比如不用创建额外的栈帧, 为其他的优化建立基础.
- 逃逸分析. 
