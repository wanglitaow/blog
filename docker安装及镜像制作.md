@[TOC](Docker安装及镜像制作)
# 安装docker

## 基本配置

``` 
yum install -y vim lrzsz git
setenforce 0
vim /etc/selinux/config
SELINUX=disabled
systemctl stop firewalld
systemctl disable firewalld
```
## centos
``` 
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine -y  移除旧版本
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum list docker-ce --showduplicates | sort -r
yum install -y docker-ce-18.09.5
systemctl restart docker

yum update -y nss curl libcurl
curl -L https://github.com/docker/compose/releases/download/1.20.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
## ubuntu

``` 
apt-get update
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```
## Docker配置及基本使用

``` 
vim ~/.bashrc
alias dops='docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t"'
alias dopsa='docker ps -a --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t"'
alias dolo='docker logs -ft'


vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://3gki6pei.mirror.aliyuncs.com"],
  "storage-driver":"devicemapper"
}
systemctl daemon-reload
systemctl restart docker.service

service docker stop        删除所有镜像
rm -rf /var/lib/docker        
systemctl start docker.service
--restart=always    设置重启docker自动启动容器
docker update --restart=always es-node2
docker run -itd myimage:test /bin/bash -c "命令1;命令2"        启动容器自动执行命令

docker export registry > /home/registry.tar        将容器打成tar包
scp /home/registry.tar root@192.168.2.7:/root
cat ~/registry.tar | docker import - registry/2.5    将tar包打成镜像
docker save jdk1.8 > /home/java.tar.gz 导出镜像
docker load < /home/java.tar.gz 导入镜像
docker stop sentine1 | xargs docker rm
docker rm -f redis-slave1|true

docker logs -f -t --since="2018-02-08" --tail=100 CONTAINER_ID	查看指定时间后的日志，只显示最后100行
docker logs --since 30m CONTAINER_ID	查看最近30分钟的日志
/var/lib/docker/containers/contain id	下rm -rf *.log		删除docker日志

```
> 进入https://homenew.console.aliyun.com/ 搜索容器镜像服务，进入侧边栏的镜像加速器获取自己的Docker加速镜像地址。
# 安装Registry私服
## 方案1

``` 
docker run -di --name=registry -p 5000:5000 docker.io/registry
```
访问 http://192.168.2.5:5000/v2/_catalog

## 方案2

vim /usr/local/docker/registry/docker-compose.yml
``` 
version: '3.1'
services:
  registry:
    image: registry
    restart: always
    container_name: registry
    ports:
      - 5000:5000
    volumes:
      - ./data:/var/lib/registry


  frontend:
    image: konradkleine/docker-registry-frontend:v2
    container_name: registry-frontend
    restart: always
    ports:
      - 184:80
    volumes:
      - ./certs/frontend.crt:/etc/apache2/server.crt:ro
      - ./certs/frontend.key:/etc/apache2/server.key:ro
    environment:
      - ENV_DOCKER_REGISTRY_HOST=192.168.2.5
      - ENV_DOCKER_REGISTRY_PORT=5000
```
使用<kbd>docker-compose up -d</kbd>启动registry容器，http://192.168.2.5:184/ 访问私有镜像库

``` 
curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" 192.168.2.7:5000/v2/ht-micro-record-service-user/manifests/1.0.0-SNAPSHOT    查看镜像Etag
curl -I -H "Accept: application/vnd.docker.distribution.manifest.v2+json" 192.168.2.5:5000/v2/ht-micro-record-commons/manifests/1.0.0-SNAPSHOT  查看镜像Etag
curl -i -X DELETE 192.168.2.5:5000/v2/codewj-redis-cluster/manifests/sha256:d6d6fad1ac67310ee34adbaa72986c6b233bd713906013961c722ecb10a049e5    删除codewj-redis-cluster:latest镜像
curl -I -X DELETE http://192.168.2.7:5000/v2/ht-micro-record-service-user/manifests/sha256:a6d4e02fa593f0ae30476bda8d992dcb0fc2341e6fef85a9887444b5e4b75a04 删除user镜像

docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml        垃圾回收blob
docker exec registry rm -rf /var/lib/registry/docker/registry/v2/repositories/codewj-redis-cluster 强制删除
docker exec registry rm -rf /var/lib/registry/docker/registry/v2/repositories/ht-micro-record-service-user  强制删除
```

# JDK镜像制作
## 环境配置

vim /etc/docker/daemon.json
``` 
{
  "registry-mirrors": ["https://dhq9bx4f.mirror.aliyuncs.com"],
  "storage-driver": "devicemapper",
  "insecure-registries":["192.168.2.5:5000"]
} 
vim /lib/systemd/system/docker.service            开放访问
ExecStart 新增 -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \
systemctl daemon-reload
systemctl restart docker
docker start registry
```
## 制作镜像

``` 
mkdir -p /usr/local/dockerjdk8 && cd /usr/local/dockerjdk8
```
sz jdk-8u60-linux-x64.tar.gz 传到该目录下
vim Dockerfile

``` 
#依赖镜像名称和ID
FROM docker.io/centos:7
#指定镜像创建者信息
MAINTAINER OneJane
#切换工作目录
WORKDIR /usr
RUN mkdir /usr/local/java
#ADD 是相对路径jar,把java添加到容器中
ADD jdk-8u60-linux-x64.tar.gz /usr/local/java/
#配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_60
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH
```
<kbd>docker build -t='jdk1.8' .</kbd>
## 上传到私服

``` 
docker commit 6ea1085dfc2a pxc:v1.0        将镜像保存本地
docker commit -m  "容器说明"   -a  "OneJane"   [CONTAINER ID]  [给新的镜像命名]        将容器打包成镜像


docker tag jdk1.8 192.168.2.5:5000/jdk1.8
docker push 192.168.2.5:5000/jdk1.8        将镜像推到仓库
```
# Redis镜像制作
vim entrypoint.sh

``` bash
#!/bin/sh
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

``` 
#端口
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

```
#基础镜像
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
<kbd>docker build -t codewj/redis-cluster:1.0 . </kbd>
## 上传到私服

```
docker tag codewj/redis-cluster:1.0 192.168.2.5:5000/codewj-redis-cluster
docker push 192.168.2.5:5000/codewj-redis-cluster
```
# Redis
docker run -d --privileged=true -p 6379:6379 -v $PWD/redis.conf:/etc/redis/redis.conf -v $PWD/data:/data --name redis redis redis-server /etc/redis/redis.conf --appendonly yes
http://download.redis.io/redis-stable/redis.conf

``` yaml
bind 0.0.0.0
port 6379
daemonize no
appendonly yes
protected-mode no
```

# 手动部署
```		

mvn clean install deploy docker:build -DpushImage
docker run -di --name=panchip -v /tmp/saas:/tmp/saas --net=host 192.168.2.7:5000/panchip:1.0.0-SNAPSHOT
docker export panchip > /home/panchip.tar 	容器打成tar
cat panchip.tar | docker import - panchip   tar转镜像
```

详情见：
https://github.com/OneJane/blog