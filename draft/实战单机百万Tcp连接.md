# 实战单机百万Tcp连接

## 准备工作

### Server端服务器配置

1. 修改系统最大文件打开数。

临时性方案：`echo 1200000 > /proc/sys/fs/file-max`

永久性方案：在`/etc/sysctl.conf`中添加`fs.file-max = 1200000`，设置完成后执行命令：sudo sysctl -p，重新装载配置使其生效。

查看当前系统使用的打开文件描述符数：`cat /proc/sys/fs/file-nr`。

``` sh
 ~/WorkSpace/Java/vert.x   sc  cat /proc/sys/fs/file-nr 
11069	0	2000000
```

执行命令会输出三个数值，第一个表示当前系统已分配使用的文件数，第二个数为分配后已释放的（目前已不再使用），第三个数是系统最大文件数，等于file-max值。

2. 修改进程最大文件打开数。

这个参数默认值比较小，我的ubuntu14.04上默认是1024个。如果我们创建大量的链接，当超过这个值时将会抛出：[Too many open files](https://gist.github.com/focusj/bd3b6165364764d1b70814faf256c4e4)错误。

首先，查看本机的默认最大文件打开数：`ulimit -n`。如果我们想创建一百万链接的话，这个值应该设置大于1000000的值。

临时修改方案：
```
ulimit -Sn 1048576
ulimit -Hn 1048576
```
我的ubuntu 17.04在做这个修改的时候遇到一点问题，总是设置不成功，最后发现是系统bug。

永久修改方案，需要修改/etc/security/limits.conf：
```
*         hard    nofile      1048576
*         soft    nofile      1048576
root      hard    nofile      1048576
root      soft    nofile      1048576
```
`*`代表非root用户。修改之后需重启系统或者重新登录使配置生效。

设置完成之后我们可以启动进程检测当前进程的最大文件打开数：`cat /proc/${pic}/limits`:
```
 ~/WorkSpace/Java/vert.x   sc  cat /proc/27136/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             46651                46651                processes 
Max open files            1048576              1048576              files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       46651                46651                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us     
```
可以看到open files一行已经变成我们的预设值。

在修改的时候需要注意一点，soft 不能大于hard值。



1. 测试现象一：Too many open files。



查看系统最大文件打开数设置：
cat /proc/sys/fs/file-nr


修改系统值：
sudo vi /etc/sysctl.conf

fs.file-max = 1020000
net.ipv4.ip_conntrack_max = 1020000
net.ipv4.netfilter.ip_conntrack_max = 1020000

解决ubuntu 17.04中总是不能设置成功的问题：https://superuser.com/questions/1200539/cannot-increase-open-file-limit-past-4096-ubuntu



以上做法只是临时，系统下次重启，会还原。 更为稳妥的做法是修改/etc/sysctl.conf文件，增加一行内容

net.ipv4.ip_local_port_range= 1024 65535

每个IP只能创建大概6万多个connection（我本机测试64500个链接）。



