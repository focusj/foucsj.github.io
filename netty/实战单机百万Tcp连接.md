# 实战单机百万Tcp连接

## 准备工作

### Server端服务器配置

1.修改系统最大文件打开数。

临时性方案：`echo 1200000 > /proc/sys/fs/file-max`

永久性方案：在`/etc/sysctl.conf`中添加`fs.file-max = 1200000`，设置完成后执行命令：sudo sysctl -p，重新装载配置使其生效。

查看当前系统使用的打开文件描述符数：`cat /proc/sys/fs/file-nr`。

``` sh
 ~/WorkSpace/Java/vert.x   sc  cat /proc/sys/fs/file-nr 
11069	0	2000000
```

执行命令会输出三个数值，第一个表示当前系统已分配使用的文件数，第二个数为分配后已释放的（目前已不再使用），第三个数是系统最大文件数，等于file-max值。

2.修改进程最大文件打开数。

这个参数默认值比较小，我的ubuntu14.04上默认是1024个。如果我们创建大量的链接，当超过这个值时将会抛出：[Too many open files](https://gist.github.com/focusj/bd3b6165364764d1b70814faf256c4e4)错误。

首先，查看本机的默认最大文件打开数：`ulimit -n`。如果我们想创建一百万链接的话，这个值应该设置大于1000000的值。

临时修改方案，这种方案及时生效，但是重启系统后配置丢失。

```bash
ulimit -Sn 1200000
ulimit -Hn 1200000
```

永久修改方案，需要修改/etc/security/limits.conf，`*`代表非root用户。如果只想为某一个用户设置的话*换成用户名即可。

```bash
*         hard    nofile      1200000
*         soft    nofile      1200000
root      hard    nofile      1200000
root      soft    nofile      1200000
```

保存修改需重启系统或者重新登录使配置生效。

我们可以启动一个进程来看一下它的最大文件打开数：`cat /proc/${pic}/limits`。以下数据是我启动的一个vertx Server进程：

```bash
 ~/WorkSpace/Java/vert.x   sc  cat /proc/27136/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             46651                46651                processes 
Max open files            1200000              1200000              files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       46651                46651                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us     
```

可以看到open files一行已经变成我们的预设值。在修改的时候需要注意一点ulimit的之不能超过系统的max file size。

我的ubuntu 17.04在做这个修改的时候遇到一点问题，设置完成后重启系统，启动程序发现最大打开文件总是4096。折腾了半天发现是ubuntu系统的[bug](https://bugs.launchpad.net/ubuntu/+source/lightdm/+bug/1627769)。解决方案来自[这里](https://superuser.com/questions/1200539/cannot-increase-open-file-limit-past-4096-ubuntu)。

3.TCP协议栈优化

### Client端服务器配置

1.修改端口范围。

linux服务器上端口范围是0～65535，其中0～1024是给系统保留的端口范围。当客户端建立链接的时候会自动的选择一个本地可用的端口号，为了能够最大限度的利用机器上的端口，我们需要修改端口范围。

通过命令`cat /proc/sys/net/ipv4/ip_local_port_range`我们可以查看当前的端口范围配置。

修改的话我们在/etc/sysctl.conf中添加这行：`net.ipv4.ip_local_port_range= 1024 65535`，然后重载配置。这样的话一个client服务器可以创建六万多个链接（我本机测试64500个链接）。

即便是这样我们也需要大概17台客户端测试机，这我上哪搞去啊。只能通过虚拟IP来弄了（系统知识严重匮乏，网上找了很长时间才搞定）。接下来我们来配置虚拟IP。配置虚拟IP我们可以通过ifconfig来搞。比如我们直接执行命令：`sudo ifconfig eth0:0 192.168.199.100 netmask 255.255.255.0`。eth0是真实网卡的名字，eth0:0是虚拟网卡的名字，后边是网卡绑定的IP。如果要想局域网访问，这个IP要和你真实网卡上的IP在同一网段。本地访问的话就随意了。

还有另外一种方式是修改/etc/network/interfaces配置文件。我们添加一下条目：
```
auto eth0:0
iface eth0 inet static
address 192.168.1.201
netmask 255.255.255.0
broadcast 192.168.1.255
gateway 192.168.1.0
```
添加完成后，重启network service：`sudo /etc/init.d/networking restart`。通过执行ifconfig便可看到这个虚拟IP。

为啥一个server不能创建超过65536个链接呢？这得需要先了解一下系统是如何标识一个网络套接字（socket）的。当系统创建一个网络套接字，会以{源IP地址，源端口，目的IP地址，目的端口}这样一个四元组来唯一表示这个链接。假设一个server拥有一个IP地址，而端口号最多就是65536个（极限）因此最大链接数也就固定了。如果我们在一个server上维持创建100M个链接，粗略估算一个IP 60000个链接，我们将需要17个IP地址。

## 开始测试

测试程序的都是用Vert.x实现的，源码[在此](https://github.com/focusj/vertx-tcp)。代码是非常简单的，没必要说了。看看在测试过程中遇到的一些问题。

1.当连接增加到23万多的时候，服务器hang住了，不在接受任何链接。

```sh
...
231000 connections connects in...
232000 connections connects in...
```

通过dmesg查看系统日志发了一些线索：

```sh
[13662.489289] TCP: too many orphaned sockets
[13662.489299] TCP: too many orphaned sockets
[13662.489304] TCP: too many orphaned sockets
[13662.489311] TCP: too many orphaned sockets
[13662.489317] TCP: too many orphaned sockets
[13662.489323] TCP: too many orphaned sockets
[13662.489330] TCP: too many orphaned sockets
[13662.489335] TCP: too many orphaned sockets
[13662.489341] TCP: too many orphaned sockets
[13662.489348] TCP: too many orphaned sockets
```

网上搜了一下，这是系统耗光了socket内存，导致新连接进来时无法分配内存。需要调整一下tcp socket参数。在tcp_mem三个值分别代表low，pressure，high三个伐值。

low：当TCP使用了低于该值的内存页面数时，TCP不会考虑释放内存。
pressure：当TCP使用了超过该值的内存页面数量时，TCP试图稳定其内存使用，进入pressure模式，当内存消耗低于low值时则退出pressure状态。
high：允许所有tcp sockets用于排队缓冲数据报的页面量，当内存占用超过此值，系统拒绝分配socket，后台日志输出"TCP: too many of orphaned sockets"。

打开/etc/sysctl.conf，加入以下配置：

```sh
net.ipv4.tcp_mem = 786432 2097152 3145728 
net.ipv4.tcp_rmem = 4096 4096 6291456
net.ipv4.tcp_wmem = 4096 4096 6291456
```

tcp_mem 中的单位是页，1页=4096字节。所以我们设置的最大tcp 内存是12G。tcp_rmem，tcp_wmem单位是byte，所以最小socket读写缓存是4k。

2.server 又卡住不创建链接了。
```
236000 connections connects in...
237000 connections connects in...
238000 connections connects in...
```

再次查看系统异常信息。

```
[ 1465.133062] nf_conntrack: nf_conntrack: table full, dropping packet
[ 1465.133066] nf_conntrack: nf_conntrack: table full, dropping packet
[ 1470.134845] net_ratelimit: 23807 callbacks suppressed
[ 1470.134846] nf_conntrack: nf_conntrack: table full, dropping packet
[ 1470.154131] nf_conntrack: nf_conntrack: table full, dropping packet
[ 1470.154138] nf_conntrack: nf_conntrack: table full, dropping packet
[ 1470.161674] nf_conntrack: nf_conntrack: table full, dropping packet
```
nf_conntrack就是linux NAT的一个跟踪连接条目的模块，nf_conntrack模块会使用一个哈希表记录TCP通讯协议的创建的链接记录，当这个哈希表满了的时候，便会导致：`nf_conntrack: table full, dropping packet`错误。那我们接下来修改一下这个值，还是打开/etc/sysctl.conf文件。
加入以下记录：
```sh
net.netfilter.nf_conntrack_max = 1200000
net.netfilter.nf_conntrack_buckets = 150000
```
这两个值表示conntrack的最大值，以及哈希的桶数。

还有一个关于JVM的调优，这个也要注意，刚开始创建几万个的时候没什么影响，但是上20万之后GC影响逐渐显著。

### 参考
[使用四种框架分别实现百万websocket常连接的服务器](http://colobu.com/2015/05/22/implement-C1000K-servers-by-spray-netty-undertow-and-node-js/)
[高性能网络编程7--tcp连接的内存使用](http://blog.csdn.net/russell_tao/article/details/18711023)
[Conntrack Tuning](https://wiki.khnet.info/index.php/Conntrack_tuning)