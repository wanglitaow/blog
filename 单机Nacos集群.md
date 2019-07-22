@[TOC]

# 环境搭建
``` 
yum update -y nss curl libcurl
curl -L https://github.com/docker/compose/releases/download/1.20.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
git clone https://github.com/nacos-group/nacos-docker.git 
cd nacos-docker
映射cluster-hostname启动脚本   - ../docker-startup.sh:/home/nacos/bin/docker-startup.sh
修改docker-startup.sh中环境变量   JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
vim ~/.bashrc
alias dofo='docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Image}}\t"'
alias dolo='docker logs -ft'
source ~/.bashrc
#docker-compose up -d
#docker-compose logs -ft
#docker-compose down
docker-compose -f example/cluster-hostname.yaml up -d
docker stop nacos1 nacos2 nacos3
docker start nacos1
docker start nacos2
docker start nacos3
docker-compose -f example/cluster-hostname.yaml down
```
http://192.168.2.5:8848/nacos  http://192.168.2.5:8849/nacos  http://192.168.2.5:8850/nacos           nacos/nacos
# Nginx负载均衡

``` 
mkdir /usr/local/docker/nginx/nacos -p
vim n1.conf
        upstream nacos { # 配置负载均衡
                server 192.168.2.5:8848;
                server 192.168.2.5:8849;
                server 192.168.2.5:8850;
        }
        server {
        listen       8841;
        server_name  192.168.2.5;
        location / {
            proxy_pass   http://nacos;
            index  index.html index.htm;
        }

vim n2.conf
        upstream nacos { # 配置负载均衡
                server 192.168.2.5:8848;
                server 192.168.2.5:8849;
                server 192.168.2.5:8850;
        }
        server {
        listen       8842;
        server_name  192.168.2.5;
        location / {
            proxy_pass   http://nacos;
            index  index.html index.htm;
        }
```
> docker run -it -d --name nginx2 -v /usr/local/docker/nginx/nacos/n2.conf:/etc/nginx/nginx.conf --net=host --privileged nginx
docker run -it -d --name nginx1 -v /usr/local/docker/nginx/nacos/n1.conf:/etc/nginx/nginx.conf --net=host --privileged nginx

http://192.168.2.5:8841/nacos/#/login   http://192.168.2.5:8842/nacos/#/login  docker pause nacos1 测试页面维持访问

# Keepalive双机热备

``` 
docker exec -it nginx1 bash
mv /etc/apt/sources.list sources.list.bak
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
apt-get update
apt-get install gnupg -y --allow-unauthenticated
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
apt-get update
apt-get install keepalived vim -y
vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface docker0         # 填写docker网卡名
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.17.0.100            # 定义该网卡下的虚拟ip地址段地址
    }
} 
service keepalived start

docker exec -it nginx2 bash
mv /etc/apt/sources.list sources.list.bak
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
apt-get update
apt-get install gnupg -y --allow-unauthenticated
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
apt-get update
apt-get install keepalived vim -y
vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface docker0         # 填写docker网卡名
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        172.17.0.100           # 定义虚拟ip地址段地址
    }
} 
service keepalived start

```
由于docker内的虚拟ip不能被外界访问借助宿主机keepalived映射外网可以访问的虚拟ip，
``` 
exit
yum install keepalived -y
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface enp1s0f0            # 宿主机网卡
    virtual_router_id 51            # 保持一致
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
           192.168.2.155            # 宿主机虚拟ip
    }
}




virtual_server 192.168.2.155 183 {        # 宿主机虚拟ip及开放端口
    delay_loop 3
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP


    real_server 172.17.0.100 8841 {         # nginx1容器虚拟ip及开放nacos端口
        weight 1
    }
}




virtual_server 192.168.2.155 183 {
    delay_loop 3
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP


    real_server 172.17.0.100 8842 {        # nginx2容器虚拟ip及开放nacos端口
        weight 1
    }
}

#apt-get --purge remove keepalived -y
#/sbin/ip addr del 192.168.12.100/32 dev enp1s0f0        删除虚拟ip

```


> 如docker0没有则重启docker，再次检查创建
systemctl daemon-reload
systemctl restart docker



访问 http://192.168.2.155:183/nacos/#/login nacos nacos

详情见：
https://github.com/OneJane/blog