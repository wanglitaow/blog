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

详情见：
https://github.com/OneJane/blog
https://www.jianshu.com/u/b2a63c970be4