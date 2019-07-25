@[TOC]

curl -L <https://github.com/coreos/etcd/releases/download/v2.2.1/etcd-v2.2.1-linux-amd64.tar.gz> -o etcd-v2.2.1-linux-amd64.tar.gz
# etcd
## 集群

``` linux
tar xzvf etcd-v2.2.1-linux-amd64.tar.gz && cd etcd-v2.2.1-linux-amd64

{NODE_NAME}:etcd节点名称，需要和命令中的-initial-cluster的对应的{NODE1_NAME}或{NODE2_NAME}对应
{NODE_IP}/{NODE1_IP}/{NODE2_NAME}：节点的IP

./etcd -name {NODE_NAME} -initial-advertise-peer-urls [http://{NODE_IP}:2380](http://NODE_IP:2380) \
-listen-peer-urls <http://0.0.0.0:2380> \
-listen-client-urls [http://0.0.0.0:2379,http://127.0.0.1:4001](http://0.0.0.0:2379,http:/127.0.0.1:4001) \
-advertise-client-urls <http://0.0.0.0:2379> \
-initial-cluster-token etcd-cluster \
-initial-cluster {NODE1_NAME}=http://{NODE1_IP}:2380,{NODE2_NAME}=http://{NODE2_IP}:2380 \
-initial-cluster-state new
```


## 单机

``` linux
tar xzvf etcd-v2.2.1-linux-amd64.tar.gz && cd etcd-v2.2.1-linux-amd64
nohup ./etcd --advertise-client-urls 'http://192.168.3.226:2379' --listen-client-urls 'http://0.0.0.0:2379' &
./etcdctl member list		查看启动情况
./etcdctl mk name OneJane
./etcdctl get name
其他主机 
./etcdctl -endpoint http://192.168.3.226:2379 get name
./etcdctl -endpoint http://192.168.3.226:2379 mk age 22
```
# swarm
## docker配置 
192.168.3.224

``` crystal
vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.3.224:2379 --cluster-advertise=192.168.3.224:2375
systemctl daemon-reload
service docker start
```

192.168.3.227

``` crystal
vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store=etcd://192.168.3.224:2379 --cluster-advertise=192.168.3.227:2375
systemctl daemon-reload
service docker start
```
## overlay

``` x86asm
192.168.3.224
docker swarm init --advertise-addr 192.168.3.224
192.168.3.227
docker swarm join --token SWMTKN-1-4qdkodh0g0c73iw5oehhn4rmsxxxca1cdfushtujspjsn1i827-3x1zyoo4tm4d4qm9x8f4o658q 192.168.3.227:2377
192.168.3.224
docker network create --driver overlay --attachable overnet
docker run -itd --name=worker-1 --net=overnet ubuntu
docker exec worker-1 apt-get update
docker exec worker-1 apt-get install net-tools
docker exec worker-1 ifconfig    10.0.0.5
192.168.3.227
docker run -itd --name=worker-2 --net=overnet ubuntu
docker exec worker-2 apt-get update
docker exec worker-2 apt-get install net-tools
docker exec worker-2 apt-get install -y inetutils-ping
docker exec worker-2 ifconfig
docker exec worker-2 ping 10.0.0.5
```


# PXC
192.168.3.224

``` haml
docker volume create v01
docker run -d \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-v v01:/var/lib/mysql \
--privileged \
--name=node1 \
--net=overnet \
percona/percona-xtradb-cluster:5.7
```

192.168.3.227

``` haml
docker volume create v02
docker run -d \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-e CLUSTER_JOIN=node1 \
-v v02:/var/lib/mysql \
--name=node2 \
--net=overnet \
percona/percona-xtradb-cluster:5.7
```

详情见：
https://github.com/OneJane/blog