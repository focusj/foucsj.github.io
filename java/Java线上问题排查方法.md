# Java线上问题排查方法

我们经常遇到的Java线上问题可能有以下几种:

- Java进程占用CPU过高. 导致这个问题的原因可能有程序实现错误, 比如死循环, 导致cpu占用过高；还有死锁也可能导致cpu占用过高；还有可能是垃圾收集线程频繁, 占用了大量的cpu.
- 内存占用过多/OOM问题. 这个问题主要的原因是内存泄漏, 比如使用了全局的内存缓存, 可能导致这个问题.
- 垃圾收集日志中反馈的一些问题. 

## CPU过高的问题

排查这个问题的话, 我们可能会用到以下这些工具: jps, top, jstack这些工具. 

jps能够查询到当前启动的Java进程, 使用`-l`参数可以显示出启动类的全路径. 
top能够查看进程当前的状态信息, 使用`-p`参数可以查看某个进程的信息. `-H`可以显示出这个进程开启的Thread. 
jstack可以打印出当前运行的Java进程的堆栈信息. 

下边说一下具体应该怎么查. 首先我们可以通过top和jps查出当前占用cpu的java进程号. 然后我们通过过top可以看到java进程开启的所有线程, 找到占用cpu最高的那条线程. 因为jstack中显示的是十六进制的进程号, 所以, 我们需要把刚才找到的进程号转成十六进制. 可以使用命令`printf "%x\n" $pid`. 然后我们需要执行jstack命令来查看线程的堆栈信息查看当前线程正在干什么.

我们具体的来分析一个例子, 这是我们的程序:
```java
public class test {
    public static void main(String[] args) {
        int i = 0;
        while (true) {
            i++;
            i--;
        }
    }
}
```

这个程序包含一个死循环, 所以cpu马上飙起来. 我们首先通过jps找到这个程序的进程号: 

```sh
 ~/3stone/ico-api   master  jps -l
10721 com.intellij.idea.Main
10885 org.jetbrains.idea.maven.server.RemoteMavenServer
19845 org.jetbrains.jps.cmdline.Launcher
19846 test
19886 sun.tools.jps.Jps
```

拿到进程号之后我们可以通过top命令来查看到底是哪条线程除了问题:
```sh
 ✘  ~/3stone/ico-api   master  jps -l
10721 com.intellij.idea.Main
20593 org.jetbrains.jps.cmdline.Launcher
20595 test
20660 sun.tools.jps.Jps
10885 org.jetbrains.idea.maven.server.RemoteMavenServer
 ~/3stone/ico-api   master  top -p 20595

top - 15:12:07 up 1 day, 18 min,  5 users,  load average: 0.52, 0.41, 0.44
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.9 us,  1.7 sy,  0.0 ni, 92.3 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 11995048 total,  4361108 free,  4883052 used,  2750888 buff/cache
KiB Swap: 12267516 total, 12267516 free,        0 used.  6630484 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
20595 focusj    20   0 5541704  38640  19876 S 100.0  0.3   0:12.85 java
top - 15:12:24 up 1 day, 18 min,  5 users,  load average: 0.63, 0.44, 0.45
Threads:  16 total,   1 running,  15 sleeping,   0 stopped,   0 zombie
%Cpu(s): 26.3 us,  0.4 sy,  0.0 ni, 73.1 id,  0.2 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem : 11995048 total,  4360208 free,  4883896 used,  2750944 buff/cache
KiB Swap: 12267516 total, 12267516 free,        0 used.  6629632 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
20600 focusj    20   0 5541704  38640  19876 R 99.9  0.3   0:30.22 java
20595 focusj    20   0 5541704  38640  19876 S  0.0  0.3   0:00.00 java
20603 focusj    20   0 5541704  38640  19876 S  0.0  0.3   0:00.00 java
```

通过top输出的信息我们可以看到20600占用的cpu太高了, 这指定是有问题的. 我们把这个进程号转换成16进制: 

```sh
 ~/3stone/ico-api   master  printf "%x\n" 20595
5073
```

然后我们查看进程的堆栈信息:

```sh
 ~/3stone/ico-api   master  jstack 20595 | grep -A 20 5078
"main" #1 prio=5 os_prio=0 tid=0x00007f325c011000 nid=0x5078 runnable [0x00007f3262ace000]
   java.lang.Thread.State: RUNNABLE
	at test.main(test.java:6)
```

通过查看堆栈信息我们可以去检查程序的第6行附近代码.

还有一种情况是垃圾回收线程导致的cpu过高, 这种情况比较复杂, 需要分析gc log, 然后还要分析内存的使用情况, 所以不在这分析了. 

## 内存占用过多

Java语言不需要我们手动的管理内存了, 但是如果不注意仍然可能会出现OOM的问题. 对于OOM的问题, 最有用的依据是当前进程的内存使用状况, 以及GC log. 所以对于线上程序最好能够在启动的时候打印出GC log: `-Xloggc:/dev/shm/gc-myapplication.log`. 

查询内存问题我们需要用到以下工具jstat, jstat可以查询jvm运行的一些数据. 比如我们想要查询gc信息可以执行以下命令:

```js
 ~/3stone/ico-api   master  jstat -gccause 10721
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC
 44.14   0.00  66.14  71.01  93.74  90.70    362    6.486    25    3.331    9.817 Allocation Failure   No GC 

 ~/3stone/ico-api   master  jstat -gc 10721
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
14848.0 14848.0 6553.4  0.0   119168.0 75897.1   297556.0   211290.7  342864.0 321394.9 50088.0 45431.6    362    6.486  25      3.331    9.817
```

以下对这些参数进行说明:
```
S0C、S1C、S0U、S1U：Survivor 0/1区容量（Capacity）和使用量（Used）
EC、EU：Eden区容量和使用量
OC、OU：年老代容量和使用量
PC、PU：永久代容量和使用量
YGC、YGT：年轻代GC次数和GC耗时
FGC、FGCT：Full GC次数和Full GC耗时
GCT：GC总耗时
```

还有一个工具jmap, 这个可以查看当时的堆内存情况, 以及垃圾手机算法.
```sh
 ~  jmap -heap 27155
Attaching to process ID 27155, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.144-b01

using thread-local object allocation.
Parallel GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 3072327680 (2930.0MB)
   NewSize                  = 63963136 (61.0MB)
   MaxNewSize               = 1023934464 (976.5MB)
   OldSize                  = 128974848 (123.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 401080320 (382.5MB)
   used     = 284107192 (270.9457321166992MB)
   free     = 116973128 (111.55426788330078MB)
   70.83548552070567% used
From Space:
   capacity = 15204352 (14.5MB)
   used     = 14860464 (14.172042846679688MB)
   free     = 343888 (0.3279571533203125MB)
   97.73822652882544% used
To Space:
   capacity = 19922944 (19.0MB)
   used     = 0 (0.0MB)
   free     = 19922944 (19.0MB)
   0.0% used
PS Old Generation
   capacity = 121110528 (115.5MB)
   used     = 25487824 (24.307083129882812MB)
   free     = 95622704 (91.19291687011719MB)
   21.045093618946158% used

23078 interned Strings occupying 2932240 bytes.
```

使用jmap还可以打印堆内存数目, 大小统计的直方图: 
```
 ~  jmap -histo:live 27769 | more

 num     #instances         #bytes  class name
----------------------------------------------
   1:         88123       13046328  [C
   2:         31343        2758184  java.lang.reflect.Method
   3:         86750        2082000  java.lang.String
   4:          7407        1463152  [B
   5:         45167        1445344  java.util.concurrent.ConcurrentHashMap$Node
   6:          8811         982720  java.lang.Class
   7:         22133         885320  java.util.LinkedHashMap$Entry
   8:         18164         844520  [Ljava.lang.Object;
   9:         11823         662088  java.util.LinkedHashMap
  10:          9689         656176  [Ljava.util.HashMap$Node;
  11:         20493         655776  java.lang.ref.WeakReference
  12:         24154         565112  [Ljava.lang.Class;
  13:           280         462144  [Ljava.util.concurrent.ConcurrentHashMap$Node;
  14:         12673         405536  java.util.HashMap$Node
  15:         15624         374976  org.springframework.core.MethodClassKey
  16:         14314         343536  java.util.ArrayList
  17:          8277         331080  java.lang.ref.SoftReference
  18:          4525         325800  java.lang.reflect.Field
  19:         15685         250960  java.lang.Object
  20:          9817         235608  java.beans.MethodRef
  21:          4061         219536  [I
  22:          3917         219352  java.beans.MethodDescriptor
```

还可以使用jmap把进程内存使用情况dump到文件中，再用jhat分析查看。jmap进行dump命令格式如下：
```
jmap -dump:format=b,file=/tmp/dump.dat 21711

jhat -port 9998 /tmp/dump.dat
```

通过使用这些工具, 以及配合垃圾收集日志可以大概找出到底是哪个类占用了内存, 然后结合程序, 定位到问题.