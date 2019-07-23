@[TOC]

redis-sentinel，只有一个master，各实例数据保持一致；
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563845335877.png)
> 单个redis-sentinel进程来监控redis集群是不可靠的，由于redis-sentinel本身也有single-point-of-failure-problem(单点问题)，当出现问题时整个redis集群系统将无法按照预期的方式切换主从。官方推荐：一个健康的集群部署，至少需要3个Sentinel实例。另外，redis-sentinel只需要配置监控redis master，而集群之间可以通过master相互通信。


|  name   |   ip  |   port  |
| --- | --- | --- |
|  redis-master   |   192.168.2.5 |   6300  |
|   redis-slave1  |   192.168.2.7  |  6301   |
|   redis-slave2  |  192.168.2.7   |    6302 |
|   sentinel1  |   192.168.2.5 |   26000  |
|   sentinel2  |   192.168.2.7  |  26001   |
|   sentinel3  |  192.168.2.7   |   26002  |

# Redis部署
在192.168.2.5上运行

``` dsconfig
docker run -it --name redis-master --network host -d redis --appendonly yes --port 6300
```

在192.168.2.7上运行

``` lsl
docker run -it --name redis-slave1 --network host -d redis --appendonly yes --port 6301 --slaveof 192.168.2.5 6300

docker run -it --name redis-slave2 --network host -d redis --appendonly yes --port 6302 --slaveof 192.168.2.5 6300
```
# Sentinel部署

``` groovy
wget http://download.redis.io/redis-stable/sentinel.conf
```
mkdir /app/{sentine1,sentine2,sentine3}/{data,conf} -p

并复制sentinel1.conf，sentinel2.conf，sentinel3.conf

> /app
├── sentine1
│   ├── conf
│   └── data
├── sentine2
│   ├── conf
│   └── data
└── sentine3
    ├── conf
    └── data
		  
## 192.168.2.5主节点

chmod a+w -R /app/
``` 
port 26000
pidfile /var/run/redis-sentinel.pid
logfile ""
daemonize no
dir /tmp
sentinel monitor mymaster 192.168.2.5 6300 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```
创建主节点

``` stata
docker run -d --network host --name sentine1 \
-v /app/sentine1/data:/var/redis/data \
-v /app/sentine1/conf/sentinel.conf:/usr/local/etc/redis/sentinel.conf \
redis /usr/local/etc/redis/sentinel.conf --sentinel
```

## 192.168.2.7 从节点

sentinel2.conf 
``` 
port 26001
daemonize no
dir /tmp
sentinel monitor mymaster 192.168.2.5 6300 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

创建从节点1
``` stata
docker run -d --network host --name sentine2 \
-v /app/sentine2/data:/var/redis/data \
-v /app/sentine2/conf/sentinel.conf:/usr/local/etc/redis/sentinel.conf \
redis /usr/local/etc/redis/sentinel.conf --sentinel
```
## 192.168.2.7 从节点

sentinel3.conf 
``` 
port 26002
daemonize no
dir /tmp
sentinel monitor mymaster 192.168.2.5 6300 2

sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```
创建从节点2
``` stata
docker run -d --network host --name sentine3 \
-v /app/sentine3/data:/var/redis/data \
-v /app/sentine3/conf/sentinel.conf:/usr/local/etc/redis/sentinel.conf \
redis /usr/local/etc/redis/sentinel.conf --sentinel
```
# 测试

``` 
[root@centoss2 app]# redis-cli -p 26000
127.0.0.1:26000> sentinel master mymaster
```

![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563846494036.png)




详情见：
https://github.com/OneJane/blog
