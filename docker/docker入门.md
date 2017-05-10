# Docker 入门

## Install Docker on Ubuntu

- 添加docker的源

```sh
sudo apt-get -y install \
  apt-transport-https \
  ca-certificates \
  curl

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
       $(lsb_release -cs) \
       stable"

sudo apt-get update
```

- 安装docker-ce

```sh
sudo apt-get -y install docker-ce
```

- 测试

```sh
sudo docker run hello-world
```

运行这个命令的时候，docker会从docker hub下载hello-world的镜像，并启动一个全新的容器。

## 镜像加速器

配置国内镜像加速器，以备不时之需。针对两种不同的系统初始化方式systemd和upstart，配置加速器也分为两种：

- systemd

编辑/etc/systemd/system/multi-user.target.wants/docker.service，在ExecStart=这一行加上`--registry-mirror=https://xxx.mirror.aliyuncs.com`。修改完成后重新加载docker服务：

```sh
$sudo systemctl daemon-reload
$sudo systemctl restart docker
```

- upstart

找到/etc/default/docker，加入以下行`DOCKER_OPTS="--registry-mirror=https://xxx.mirror.aliyuncs.com"`，url是申请的加速器地址。然后重启docker service：`sudo service docker restart`。

## 操作镜像

- 查看和获取镜像

docker hub提供了很多镜像供大家使用，我们可以直接通过docker命令来轻松的获取它们。

搜索镜像：`docker search vertx`
拉取镜像：`docker pull vertx/vertx3`
本地镜像：`docker images`

- dangling images

有时我们执行docker images会发现一些奇怪的镜像，就像第一个。

```sh
 ✘ focusj@focusj-ThinkPad-T450  ~/3stone/mica   master  docker images           
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              c11a2fab469b        10 seconds ago      310 MB
mica-image          v1                  862f196aca79        4 hours ago         311 MB
redis               latest              e32ef7250bc1        12 days ago         184 MB
mysql               5.7                 9e64176cd8a2        12 days ago         407 MB
mysql               latest              9e64176cd8a2        12 days ago         407 MB
openjdk             8-jre               b8ce7cab8ed3        6 weeks ago         310 MB
```

这种镜像称为虚悬镜像，产生的原因是docker pull或build的时候重名，导致旧的被覆盖。通常这写虚悬镜像可以直接删掉：`docker rmi $(docker images -q -f dangling=true)`。


有时我们从docker hub下载的标准镜像并不能满足需求，需要自己做一些定制工作。这时我们可以通过编写Dockerfile来完成定制工作。

## Dockerfile

COPY 命令：拷贝文件到指定目录。源路径或文件可以是多个，也可以使用通配符。

ADD 命令：更高级的COPY命令。源路径可以是一个url，docker引擎会自动为你下载该文件。下载后的文件权限自动设置为600,如果不符合要求还需自行执行RUN命令修改权限。
如果源路径是一个tar文件，压缩格式为：gzip，bzip2，xz，ADD命令会自动把文件解压到目标路径中去。




