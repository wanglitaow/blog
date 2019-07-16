# Haproxy 负载均衡
haproxy提供负载均衡，并自动切换故障容器
vim /usr/local/docker/mysql/haproxy/haproxy.cfg 编写配置文件
``` global
    #工作目录
    chroot /usr/local/etc/haproxy
    #日志文件，使用rsyslog服务中local5日志设备（/var/log/local5），等级info
    log 127.0.0.1 local5 info
    #守护进程运行
    daemon


defaults
    log    global
    mode    http
    #日志格式
    option    httplog
    #日志中不记录负载均衡的心跳检测记录
    option    dontlognull
    #连接超时（毫秒）
    timeout connect 5000
    #客户端超时（毫秒）
    timeout client  50000
    #服务器超时（毫秒）
    timeout server  50000


#监控界面    
listen  admin_stats
    #监控界面的访问的IP和端口
    bind  0.0.0.0:8888
    #访问协议
    mode        http
    #URI相对地址
    stats uri   /dbs
    #统计报告格式
    stats realm     Global\ statistics
    #登陆帐户信息
    stats auth  admin:abc123456
#数据库负载均衡
listen  proxy-mysql
    #访问的IP和端口
    bind  0.0.0.0:185
    #网络协议
    mode  tcp
    #负载均衡算法（轮询算法）
    #轮询算法：roundrobin
    #权重算法：static-rr
    #最少连接算法：leastconn
    #请求源IP算法：source
    balance  roundrobin
    #日志格式
    option  tcplog
    #在MySQL中创建一个没有权限的haproxy用户，密码为空。Haproxy使用这个账户对MySQL数据库心跳检测
    option  mysql-check user haproxy
    server  MySQL_1 192.168.3.226:3317 check weight 1 maxconn 2000  
    server  MySQL_2 192.168.3.225:3318 check weight 1 maxconn 2000  
    #使用keepalive检测死链
    option  tcpka  
```
在两台Replication组建的mysql集群同时创建mysql_cluster的haproxy容器，形成集群。

``` docker run -itd -v /usr/local/docker/mysql/haproxy:/usr/local/etc/haproxy --name mysql_cluster --privileged --net host haproxy
docker exec -it ht-mysql-master mysql -u root -p 
drop user 'haproxy'@'%';
create user 'haproxy'@'%' IDENTIFIED BY '';
```
http://192.168.3.226:4001/dbs admin abc123456        实时查看haproxy监控页面    admin:abc123456
192.168.3.226 185 root 123456  访问数据库，与Replication数据同步一致。
# Keepalived双机热备高可用
**应用程序向宿主机65的发起请求，宿主机的Keepalived路由到docker内部的虚拟IP15。Haproxy容器内Keepalived抢占虚拟IP，接收到所有数据库请求将被转发到抢占虚拟IP的Haproxy，keepalived互相心跳检测，一旦主服务器挂了，备用服务器将有权抢到虚拟ip，再通过负载均衡分发到某一个PXC节点，并通过主从复制实现数据同步。**
![基于host无需宿主机Keeplived映射](https://www.github.com/OneJane/blog/raw/master/小书匠/1563271437190.png)
## 192.168.3.226
docker exec -it mysql_cluster bash        进入h1

``` 
mv /etc/apt/sources.list sources.list.bak
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse">>/etc/apt/sources.list
apt-get update
apt-get install gnupg -y --allow-unauthenticated
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
apt-get update
apt-get install keepalived vim -y
vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface ens34        # 宿主机192.168.3.226网卡
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {        # 虚拟ip
        192.168.3.222
    }
}
virtual_server 192.168.3.222 189 {        # 虚拟ip开放3306端口
    delay_loop 3
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
    real_server 192.168.3.226 185 {    # 分发请求到h1
        weight 1
    }
}
service keepalived start
ping 192.168.3.222 
```
## 192.168.3.225
docker exec -it mysql_cluster bash        进入h2

``` mv /etc/apt/sources.list sources.list.bak
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
echo "deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse">>/etc/apt/sources.list
apt-get update
apt-get install gnupg -y --allow-unauthenticated
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
apt-get update
apt-get install keepalived vim -y
vim /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens34        # 宿主机192.168.3.225网卡
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {        # 虚拟ip
        192.168.3.222
    }
}
virtual_server 192.168.3.222 189 {        # 虚拟ip开放189端口
    delay_loop 3
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP
    real_server 192.168.3.225 185 {    # 分发请求到h2
        weight 1
    }
}
service keepalived start
ping 192.168.3.222
```
访问192.168.3.222 189 root 123456