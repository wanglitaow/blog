@[TOC](Replication主从复制)
# 简单介绍
本文具体讲述mysql基于多机器的数据库高可用的一些解决方案。
- 主从复制：常见方案有PXC以及Replication。 Replication的主从在主库中操作，速度较快，弱一致性，单向异步，一旦stop slave将无法同步；PXC集群速度慢，强一致性，高价值数据，双向同步。
- 负载均衡：Nginx更适用于HTTP协议的应用负载，刚刚支持TCP；Haproxy提供负载，故障自动切换。
- 双机热备：Keepalived通过虚拟IP将请求分发，让抢占到虚拟IP的Haproxy通过负载分发给某一数据库节点。

# Replication
## 单机
### Master
```linux
docker run -di --name=mysql_master -p 3300:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
docker cp mysql_master:/etc/mysql/my.cnf /usr/share/mysql/my_master.cnf 
docker cp mysql_master:/etc/mysql/my.cnf /usr/share/mysql/my_slaver.cnf 
docker stop mysql_master
docker rm mysql_master
docker run -di --name=mysql_master -p 3300:3306 -e MYSQL_ROOT_PASSWORD=123456 
           -v /usr/share/mysql/my_master.cnf:/etc/mysql/my.cnf mysql:5.7.25
mkdir -p /usr/local/mysql_master
chown -R 777 /usr/local/mysql_master/
以上主要取出配置文件模板类型

vim /usr/share/mysql/my_master.cnf
basedir = /usr/local/mysql_master
port = 3306
server_id = 98
log_bin=zlinux01

docker restart mysql_master 
docker exec -it mysql_master /bin/bash 
mysql -u root -p 
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
FLUSH  PRIVILEGES;
grant replication slave on *.* to 'root'@'192.168.12.98' identified by '123456';
flush tables with read lock;
show master status;
+-----------------+----------+--------------+------------------+-------------------+
| File            | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-----------------+----------+--------------+------------------+-------------------+
| zlinux01.000004 |      154 |              |                  |                   |
+-----------------+----------+--------------+------------------+-------------------+
```
### Slave
```
docker run -di --name=mysql_slaver -p 3301:3306 -e MYSQL_ROOT_PASSWORD=123456 
           -v /usr/share/mysql/my_slaver.cnf:/etc/mysql/my.cnf mysql:5.7.25  

mkdir -p /usr/local/mysql_slaver
chown -R mysql.mysql /usr/local/mysql_slaver/

vim /usr/share/mysql/my_slaver.cnf
basedir = /usr/local/mysql_slaver
port = 3306
server_id = 89
log_bin=zlinux02

docker restart mysql_slaver 
docker exec -it mysql_slaver /bin/bash
mysql -u root -p 
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
stop slave;
change master to master_host='192.168.12.98',master_user='root',
  master_password='123456',master_port=3300,
  master_log_file='zlinux01.000004',master_log_pos=154;
start slave;
show slave status\G            
     Slave_IO_Running: Yes    Slave_SQL_Running: Yes则成功
```
单机集群在实际应用中毫无意义，仅供参考。
# 多机
## 一主多从
### Master 192.168.3.226
```
mkdir -p /usr/local/mysql_master
chown -R 777 /usr/local/mysql_master
docker run -di --name=mysql_master -p 3300:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql
docker cp mysql_master:/etc/mysql/my.cnf /usr/share/mysql/my_master.cnf        并修改
basedir = /usr/local/mysql_master
port = 3306
server_id = 98
log_bin=zlinux01
docker stop mysql_master 
docker rm mysql_master

docker run -di --name=mysql_master -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 
           -v /usr/share/mysql/my_master.cnf:/etc/mysql/my.cnf mysql:5.7.25
docker exec -it mysql_master /bin/bash
mysql -u root -p
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
FLUSH PRIVILEGES;
grant replication slave on *.* to 'root'@'192.168.3.225' identified by '123456';
flush tables with read lock;
show master status;
+-----------------+----------+--------------+------------------+-------------------+
| File            | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-----------------+----------+--------------+------------------+-------------------+
| zlinux01.000004 |     1135 |              |                  |                   |
+-----------------+----------+--------------+------------------+-------------------+
```
### Slaver 192.168.3.225
```
mkdir -p /usr/local/mysql_slaver
chown -R 777 /usr/local/mysql_slaver
docker cp mysql_master:/etc/mysql/my.cnf /usr/share/mysql/my_slaver.cnf        并修改
basedir = /usr/local/mysql_slaver
port = 3306
server_id = 89
log_bin=zlinux02
docker run -di --name=mysql_slaver -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 
               -v /usr/share/mysql/my_slaver.cnf:/etc/mysql/my.cnf mysql:5.7.25
docker exec -it mysql_slaver /bin/bash
mysql -u root -p
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
stop slave;
change master to master_host='192.168.3.226',master_user='root',
  master_password='123456',master_port=3306,
  master_log_file='zlinux01.000004',ster_logmaster_log_pos=1135;
start slave;
show slave status\G            证明主从复制实现
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```
以上即Replication的主从复制，单向复制，只可作为热备使用。
## 最终方案
### Master_1 192.168.3.226
```
/root
└── test
    └── mysql_test1
        ├── haproxy
        │   └── haproxy.cfg
        ├── log
        ├── mone
        │   ├── conf
        │   │   └── my.cnf
        │   └── data
        └── mtwo
            ├── conf
            │   └── my.cnf
            └── data

mkdir test/mysql_test1/{mone,mtwo}/{data,conf} -p
vim test/mysql_test1/mone/conf/my.cnf
[mysqld]
server_id = 1
log-bin= mysql-bin
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
read-only=0
relay_log=mysql-relay-bin
log-slave-updates=on
auto-increment-offset=1
auto-increment-increment=2
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/


vim test/mysql_test1/mysql/mtwo/conf/my.cnf
[mysqld]
server_id = 2
log-bin= mysql-bin
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
read-only=0
relay_log=mysql-relay-bin
log-slave-updates=on
auto-increment-offset=2
auto-increment-increment=2
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

scp -r test root@192.168.3.225:/root/                
docker run --name monemysql -d -p 3317:3306 -e MYSQL_ROOT_PASSWORD=root 
    -v ~/test/mysql_test1/mone/data:/var/lib/mysql 
    -v ~/test/mysql_test1/mone/conf/my.cnf:/etc/mysql/my.cnf mysql:5.7
docker exec -it monemysql mysql -u root -p                    输入root
stop slave;
GRANT REPLICATION SLAVE ON *.* to 'slave'@'%' identified by '123456';        
          创建一个slave同步账号slave，允许访问的IP地址为%，%表示通配符用来同步数据
show master status;            
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      443 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
docker inspect monemysql | grep IPA                查看容器ip
            "SecondaryIPAddresses": null,    
            "IPAddress": "172.17.0.2",
                    "IPAMConfig": null,
                    "IPAddress": "172.17.0.2",
```
### Master_2 192.168.3.225
```
docker run --name mtwomysql -d -p 3318:3306 -e MYSQL_ROOT_PASSWORD=root 
    -v ~/test/mysql_test1/mtwo/data:/var/lib/mysql 
    -v ~/test/mysql_test1/mtwo/conf/my.cnf:/etc/mysql/my.cnf mysql:5.7
docker exec -it mtwomysql mysql -u root -p        输入root
stop slave;
change master to master_host='192.168.3.226',master_user='slave',
  master_password='123456',master_log_file='mysql-bin.000003',
  master_log_pos=443,master_port=3317;
GRANT REPLICATION SLAVE ON *.* to 'slave'@'%' identified by '123456';        
                              创建一个用户来同步数据
start slave ;    启动同步
show master status;        查看状态
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      443 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```
### 双向同步
```
Master_1 192.168.3.226
stop slave;
change master to master_host='192.168.3.225',master_user='slave',
        master_password='123456',master_log_file='mysql-bin.000003',
        master_log_pos=443,master_port=3318;
start slave ;
```
> 在两个容器中查看 show slave status\G;
            Slave_IO_Running: Yes
            Slave_SQL_Running: Yes

双向验证，数据同步

详情见：
https://github.com/OneJane/blog
https://www.jianshu.com/u/b2a63c970be4