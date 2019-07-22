@[TOC]
>支撑多个master node，每个master挂载多个slave，master写对应slave读，每个master都有slave节点，若挂掉则将某个slave转为master。
与相比Sentinel（哨兵）实现的高可用，集群（cluster）更多的是强调数据的分片或者是节点的伸缩性，如果在集群的主节点上加入对应的从节点，集群还可以自动故障转移。
主从复制： 通过把这个RDB文件或AOF文件传给slave服务器，slave服务器重新加载RDB文件，来实现复制的功能！当建立一个从服务器后，从服务器会想主服务器发送一个SYNC的命令，主服务器接收到SYNC命令之后会执行BGSAVE，然后保存到RDB文件，然后发送到从服务器！收到RDB文件然后就载入到内存！

> Redis 集群中内置了 16384 个哈希槽，当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点

> 集群中所有master参与,如果半数以上master节点与master节点通信超过(cluster-node-timeout),认为当前master节点挂掉.
如果集群任意master挂掉,且当前master没有slave.集群进入fail状态,也可以理解成集群的slot映射[0-16383]不完成时进入fail状态
如果集群超过半数以上master挂掉，无论是否有slave集群进入fail状态.

# 多机集群
## 192.168.2.5
vim /usr/local/docker/redis/docker-compose.yml
```version: '3'
 
services:
  redis1:
    image: publicisworldwide/redis-cluster
    container_name: redis1
    network_mode: host
    restart: always
    volumes:
     - ./8001/data:/data
    environment:
     - REDIS_PORT=8001
  
  
  redis2:
    image: publicisworldwide/redis-cluster
    container_name: redis2
    network_mode: host
    restart: always
    volumes:
     - ./8002/data:/data
    environment:
     - REDIS_PORT=8002
  
  
  redis3:
    image: publicisworldwide/redis-cluster
    container_name: redis3
    network_mode: host
    restart: always
    volumes:
     - ./8003/data:/data
    environment:
     - REDIS_PORT=8003
```
docker-compose up -d
## 192.168.2.7
vim /usr/local/docker/redis/docker-compose.yml

``` version: '3'
 
services:
  redis1:
    image: publicisworldwide/redis-cluster
    network_mode: host
    container_name: redis4
    restart: always
    volumes:
     - ./8004/data:/data
    environment:
     - REDIS_PORT=8004
  
  
  redis2:
    image: publicisworldwide/redis-cluster
    network_mode: host
    container_name: redis5
    restart: always
    volumes:
     - ./8005/data:/data
    environment:
     - REDIS_PORT=8005
  
  
  redis3:
    image: publicisworldwide/redis-cluster
    network_mode: host
    container_name: redis6
    restart: always
    volumes:
     - ./8006/data:/data
    environment:
     - REDIS_PORT=8006
```
docker-compose up -d
# 启动方式
## 直接启动
``` docker run --rm -it inem0o/redis-trib create --replicas 1 192.168.2.5:8001 192.168.2.5:8002 192.168.2.5:8003 192.168.2.7:8004 192.168.2.7:8005 192.168.2.7:8006

docker exec -it redis1 redis-cli -h 127.0.0.1 -p 8001 -c
set a 100
cluster info
cluster nodes
```
> 自动指定master slave
## 指定master

``` docker run --rm -it inem0o/redis-trib create  192.168.2.5:8001 192.168.2.5:8002 192.168.2.5:8003

docker run --rm -it inem0o/redis-trib add-node --slave --master-id 84c3b7ecbc4933e1368a6927f26c79ecc76810b3 192.168.2.7:8004 192.168.2.5:8001
docker run --rm -it inem0o/redis-trib add-node --slave --master-id 716f11f2971e9494183937abd61f7a4baf0b3959 192.168.2.7:8005 192.168.2.5:8002 
docker run --rm -it inem0o/redis-trib add-node --slave --master-id c93060613a8f1531c82b97d97eeac402048f0b25 192.168.2.7:8006 192.168.2.5:8003


docker run --rm -it inem0o/redis-trib info 192.168.2.5:8001
docker run --rm -it inem0o/redis-trib help
```
> 若同一台宿主机，不想使用host模式同一台，也可以把network_mode去掉，但就要加ports映射。redis-cluster的节点端口共分为2种，一种是节点提供服务的端口，如6379；一种是节点间通信的端口，固定格式为：10000+6379。

``` docker-compose.yml
version: '3'


services:
redis1:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8001/data:/data
  environment:
   - REDIS_PORT=8001
  ports:
    - '8001:8001'
    - '18001:18001'
    
redis2:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8002/data:/data
  environment:
   - REDIS_PORT=8002
  ports:
    - '8002:8002'
    - '18002:18002'
    
redis3:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8003/data:/data
  environment:
   - REDIS_PORT=8003
  ports:
    - '8003:8003'
    - '18003:18003'
    
redis4:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8004/data:/data
  environment:
   - REDIS_PORT=8004
  ports:
    - '8004:8004'
    - '18004:18004'
    
redis5:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8005/data:/data
  environment:
   - REDIS_PORT=8005
  ports:
    - '8005:8005'
    - '18005:18005'
    
redis6:
  image: publicisworldwide/redis-cluster
  restart: always
  volumes:
   - /app/app/redis/8006/data:/data
  environment:
   - REDIS_PORT=8006
  ports:
    - '8006:8006'
    - '18006:18006'
```
# 自定义Redis集群
## 制作redis镜像
vim entrypoint.sh
``` #!/bin/sh
#只作用于当前进程,不作用于其创建的子进程
set -e
#$0--Shell本身的文件名 $1--第一个参数 $@--所有参数列表
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    sed -i 's/REDIS_PORT/'$REDIS_PORT'/g' /usr/local/etc/redis.conf
    chown -R redis .  #改变当前文件所有者
    exec gosu redis "$0" "$@"  #gosu是sudo轻量级”替代品”
fi

exec "$@"
```
vim redis.conf

``` #端口
port REDIS_PORT
#开启集群
cluster-enabled yes
#配置文件
cluster-config-file nodes.conf
cluster-node-timeout 5000
#更新操作后进行日志记录
appendonly yes
#设置主服务的连接密码
# masterauth
#设置从服务的连接密码
# requirepass
```
vi Dockerfile

``` #基础镜像
FROM redis
#修复时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
#环境变量
ENV REDIS_PORT 8000
#ENV REDIS_PORT_NODE 18000
#暴露变量
EXPOSE $REDIS_PORT
#EXPOSE $REDIS_PORT_NODE
#复制
COPY entrypoint.sh /usr/local/bin/
COPY redis.conf /usr/local/etc/
#for config rewrite
RUN chmod 777 /usr/local/etc/redis.conf
RUN chmod +x /usr/local/bin/entrypoint.sh
#入口
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
#命令
CMD ["redis-server", "/usr/local/etc/redis.conf"]
```
> docker build -t codewj/redis-cluster:1.0 .  
docker tag codewj/redis-cluster:1.0 192.168.2.5:5000/codewj-redis-cluster
docker push 192.168.2.5:5000/codewj-redis-cluster


## 192.168.2.5

``` version: '3'
services:
  redis1:
    image: 192.168.2.5:5000/codewj-redis-cluster
    container_name: redis1
    network_mode: host
    restart: always
    volumes:
     - ./8001/data:/data
    environment:
     - REDIS_PORT=8001




  redis2:
    image: 192.168.2.5:5000/codewj-redis-cluster
    container_name: redis2
    network_mode: host
    restart: always
    volumes:
     - ./8002/data:/data
    environment:
     - REDIS_PORT=8002




  redis3:
    image: 192.168.2.5:5000/codewj-redis-cluster
    container_name: redis3
    network_mode: host
    restart: always
    volumes:
     - ./8003/data:/data
    environment:
     - REDIS_PORT=8003
```
## 192.168.2.7

``` version: '3'
services:
  redis1:
    image: 192.168.2.5:5000/codewj-redis-cluster
    network_mode: host
    container_name: redis4
    restart: always
    volumes:
     - ./8004/data:/data
    environment:
     - REDIS_PORT=8004




  redis2:
    image: 192.168.2.5:5000/codewj-redis-cluster
    network_mode: host
    container_name: redis5
    restart: always
    volumes:
     - ./8005/data:/data
    environment:
     - REDIS_PORT=8005




  redis3:
    image: 192.168.2.5:5000/codewj-redis-cluster
    network_mode: host
    container_name: redis6
    restart: always
    volumes:
     - ./8006/data:/data
    environment:
     - REDIS_PORT=8006
```

详情见：
https://github.com/OneJane/blog