# 实战单机百万Tcp连接

## 准备工作 - Server端

1. 进程最大文件打开数，这个参数默认值是很小，如果不调整建立过多的连接时会跑：Too many open files错误。

查看本机的默认最大文件打开数：
```
ulimit -n
```

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

2. 系统最大文件打开数：

查看系统最大文件打开数：`cat /proc/sys/fs/file-max`

临时性设置：

`echo 1200000 > /proc/sys/fs/file-max`

永久性性设置：

在/etc/sysctl.conf中添加`fs.file-max = 1200000`，设置完成后执行命令：sudo sysctl -p，重新装载配置。


3. 查看当前系统使用的打开文件描述符数：cat /proc/sys/fs/file-nr。
```
 ~/WorkSpace/Java/vert.x   sc  cat /proc/sys/fs/file-nr 
11069	0	2000000
```
其中第一个数表示当前系统已分配使用的打开文件描述符数，第二个数为分配后已释放的（目前已不再使用），第三个数是系统最大文件数，等于file-max值。

