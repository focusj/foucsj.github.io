- 移除关闭的docker container： `docker rm $(docker ps -a -q)` 或 `docker rm -f $(docker ps -a | grep Exit | awk '{ print $1 }')`

- 移除所有的dangling状态images：`docker rmi $(docker images -q -f dangling=true)`

- 查看docker container的IP address：`docker inspect --format='{{.NetworkSettings.IPAddress}}' $cid`

- Docker查看某个容器的资源占用：`docker stats $cid`

- 启动redis并绑定localhost：`docker run --name fj-redis -p 127.0.0.1:6379:6379 -d redis`

- docker启动redis-cli：`docker run -it --link fj-redis:redis --rm redis redis-cli -h redis -p 6379`

- docker启动Consul: `docker run -d -p 8500:8500 --name=fx-dev-consul -e CONSUL_BIND_INTERFACE=eth0 consul agent -dev -ui -client=0.0.0.0 -bind='{{ GetPrivateIP }}'`

- docker启动MySQL：`docker run -p 3306:3306 --name fx-mysql -e MYSQL_ROOT_PASSWORD=000000 -e MYSQL_USER=dev -e MYSQL_PASSWORD=000000 -d mysql`