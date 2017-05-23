- 移除关闭的docker container： `docker rm $(docker ps -a -q)` 或 `docker rm -f $(docker ps -a | grep Exit | awk '{ print $1 }')`

- 移除所有的dangling状态images：`docker rmi $(docker images -q -f dangling=true)`

- 查看docker container的IP address：`docker inspect --format='{{.NetworkSettings.IPAddress}}' $cid`

- Docker查看某个容器的资源占用：`docker stats $cid`