@[TOC](Linux常用软件安装)

# 基本命令

``` 
du -h --max-depth=1  查看各文件夹大小
nohup sh inotify3.sh >>333.out & 后台执行脚本并把输出都指定文件 
jobs -l 查看运行的后台进程
fg 1 通过jobid将后台进程提取到前台运行
ctrl + z 将暂停当前正在运行到进程，fg放入后台运行
yum -c /etc/yum.conf --installroot=/usr/local --releasever=/  install lszrz  	安装文件到其他目录
```
## ekill

``` 
vim /usr/local/bin/ekill
ps aux | grep -e $* | grep -v grep | awk '{print $2}' | xargs -i kill {}
```
chmod a+x /usr/local/bin/ekill 通过ekill删除进程

# GitLab
## 方案1

``` 
docker run \
    --publish 1443:443 --publish 180:80 --publish 122:22 \
    --name gitlab \
    --volume /usr/local/docker/gitlab/config:/etc/gitlab \
    --volume /usr/local/docker/gitlab/logs:/var/log/gitlab \
    --volume /usr/local/docker/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce
```
## 方案2
vim /usr/local/docker/gitlab/docker-compose.yml
``` 
version: '3'
services:
    gitlab:
      image: 'twang2218/gitlab-ce-zh:10.5'
      restart: always
      hostname: '192.168.2.5'
      container_name: gitlab
      environment:
        TZ: 'Asia/Shanghai'
        GITLAB_OMNIBUS_CONFIG: |
          external_url 'http://192.168.2.5:180'
          gitlab_rails['gitlab_shell_ssh_port'] = 2222
          unicorn['port'] = 8888
          nginx['listen_port'] = 8080
      ports:
        - '180:8080'
        - '8443:443'
        - '2222:22'
      volumes:
        - /usr/local/docker/gitlab/config:/etc/gitlab
        - /usr/local/docker/gitlab/data:/var/opt/gitlab
        - /usr/local/docker/gitlab/logs:/var/log/gitlab

ERROR: error while removing network: network gitlab_default id e3f084651bcc6b6ca5d5b7fb122d0ef3aba108292989441abc82f14343fea827 has active endpoints
docker network inspect gitlab_default
docker network disconnect -f gitlab_default gitlab
docker-compose down --remove-orphans
docker-compose logs -ft gitlab
配置邮箱
vim /usr/local/docker/gitlab/config/gitlab.rb
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.163.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "m15806204096@163.com"
gitlab_rails['smtp_password'] = "codewj123456"
gitlab_rails['smtp_domain'] = "163.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false
gitlab_rails['gitlab_email_from'] = "m15806204096@163.com"
user["git_user_email"] = "m15806204096@163.com"
```
root/123456 管理区域->设置->开启注册 注册时发送确认邮件
docker-compose restart
新增ssh密钥 ssh-keygen -t rsa -C "15806204096@163.com",将.ssh的公钥加入Gitlab的SSH密钥
访问 http://192.168.2.5:180/ root 12345678 
## 方案3
### 安装git

``` 
yum –y install git
cd /usr/local
mkdir git
cd git
git init --bare learngit.git
useradd git
passwd git
chown -R git:git learngit.git
vi /etc/passwd
git:x:1000:1000::/home/git:/usr/bin/git-shell
复制客户端的ssh-keygen -t rsa -C "你的邮箱" 获得的id_rsa.pub公钥到/root/.ssh/authorized_keys和/root/.ssh/authorized_keys
```
> gitignore无效的解决方案

``` git rm -r --cached .
git add .
git commit -m '.gitignore'
git push origin master
*.cache
*.cache.lock
*.iml
*.log
**/target
**/logs
.idea
**/.project
**/.settings
```
### gitlab安装

``` 
git config --global http.sslVerify false
yum -y install curl policycoreutils openssh-server openssh-clients postfix
systemctl start sshd
systemctl start postfix
systemctl enable sshd
systemctl enable postfix
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
yum -y install gitlab-ce    太慢的话使用清华的源，yum makecache再install

vim  /etc/yum.repos.d/gitlab_gitlab-ce.repo
[gitlab-ce]
name=gitlab-ce
baseurl=http://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el6
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key

mkdir -p /etc/gitlab/ssl
openssl genrsa -out "/etc/gitlab/ssl/gitlab.example.com.key" 2048

openssl req -new -key "/etc/gitlab/ssl/gitlab.example.com.key" -out "/etc/gitlab/ssl/gitlab.example.com.csr"
Country Name (2 letter code) [XX]:cn
State or Province Name (full name) []:bj
Locality Name (eg, city) (Default City]:bj
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:gitlab.example.com
Email Address (]:admin@example.com
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password [):123456
An optional company name []:
```
### GitLab基本配置

``` 
openssl x509 -req -days 3650 -in "/etc/gitlab/ssl/gitlab.example.com.csr" -signkey "/etc/gitlab/ssl/gitlab.example.com.key" -out "/etc/gitlab/ssl/gitlab.example.com.crt"
openssl dhparam -out /etc/gitlab/ssl/dhparams.pem 2048
chmod 600 /etc/gitlab/ssl/*

vim /etc/gitlab/gitlab.rb     
将external_url 'http://gitlab.example.com'的http修改为https
将# nginx['redirect_http_to_https'] = false的注释去掉，修改为nginx['redirect_http_to_https'] = true
将# nginx['ssl_certificate'] = "/etc/gitlab/ssl/#{node['fqdn']}.crt"修改为# nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.example.com.crt"
将# nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/#{node['fqdn']}.key"修改为# nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.example.com.key"
将# nginx['ssl_dhparam'] = nil 修改为# nginx['ssl_dhparam'] = "/etc/gitlab/ssl/dhparams.pem"
　　
gitlab-ctl reconfigure
vim /var/opt/gitlab/nginx/conf/gitlab-http.conf
    在server_name下添加如下配置内容：rewrite ^(.*)$ https://$host$1 permanent;
重启gitlab，使配置生效,gitlab-ctl restart，如遇到访问错误直接等待启动完成
修改本地hosts文件    将gitlab服务器的地址添加 gitlab.example.com
初始化时修改管理员密码，root 12345678
git -c http.sslVerify=false clone https://gitlab.example.com/root/test-repo.git
git add .
#git config --global user.email "admin@example.com"
#git config --global user.name "admin"
git commit -m 'init'
git -c http.sslVerify=false pull origin master
git -c http.sslVerify=false push origin master

https://gitlab.example.com/admin/system_info        查看系统资源状态值
https://gitlab.example.com/admin/logs        其中application.log记录了git用户的记录，production.log实时查看所有的访问链接
https://gitlab.example.com/admin/users/new       创建用户
```
![创建用户1](https://www.github.com/OneJane/blog/raw/master/小书匠/创建用户1.png)
![创建用户2](https://www.github.com/OneJane/blog/raw/master/小书匠/创建用户2.png)
https://gitlab.example.com/root/test-repo/project_members 修改指定项目的成员，在项目的manage access中，修改登录密码
![用户设置1](https://www.github.com/OneJane/blog/raw/master/小书匠/用户设置1.png)
![用户设置2](https://www.github.com/OneJane/blog/raw/master/小书匠/用户设置2.png)
![用户设置3](https://www.github.com/OneJane/blog/raw/master/小书匠/用户设置3.png)

``` dev 对项目相关操作
rm -rf test-repo/
git -c http.sslVerify=false clone https://gitlab.example.com/root/test-repo.git
    dev
    12345678
cd test-repo/
git checkout -b release-1.0
git add .
git commit -m 'release-1.0'
git -c http.sslVerify=false push origin release-1.0
dev登录后create merge request
lead登录将受到release-1.0的merge申请，点击merge后可以填写comment并提交
```
![合并请求1](https://www.github.com/OneJane/blog/raw/master/小书匠/合并请求1.png)
![合并请求2](https://www.github.com/OneJane/blog/raw/master/小书匠/合并请求2.png)
 # Skywalking

``` 
mkdir /data2/skywalking
vim docker-compose.yml
version: '3.3'
services:
  elasticsearch:
    image: wutang/elasticsearch-shanghai-zone:6.3.2
    container_name: elasticsearch
    restart: always
    ports:
      - 191:9200
      - 192:9300
    environment:
      cluster.name: elasticsearch
docker-compose up -d
https://mirrors.huaweicloud.com/apache/incubator/skywalking/6.0.0-beta/apache-skywalking-apm-incubating-6.0.0-beta.tar.gz
tar zxf apache-skywalking-apm-incubating-6.0.0-beta.tar.gz
cd apache-skywalking-apm-incubating
vim apache-skywalking-apm-bin/config/application.yml        注释h2,打开es并修改clusterNodes地址
#  h2:
#    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
#    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
#    user: ${SW_STORAGE_H2_USER:sa}
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:""}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:192.168.2.7:191}
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
    # Batch process setting, refer to https://www.elastic.co/guide/en/elasticsearch/client/java-api/5.5/java-docs-bulk-processor.html
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:2000} # Execute the bulk every 2000 requests
    bulkSize: ${SW_STORAGE_ES_BULK_SIZE:20} # flush the bulk every 20mb
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10} # flush the bulk every 10 seconds whatever the number of requests
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2} # the number of concurrent requests
apache-skywalking-apm-incubating/webapp/webapp.yml
port 193
./apache-skywalking-apm-incubating/bin/startup.sh
```
# Sentinel

``` 
cd /data2/ && git clone https://github.com/alibaba/Sentinel.git
cd Sentinel/
mvn clean package -DskipTests
cd sentinel-dashboard/target/
nohup java -Dserver.port=190 -Dcsp.sentinel.dashboard.server=localhost:190 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar &
```
# 安装黑体字体
``` dts
yum  -y  install  fontconfig
fc-list :lang=zh
cd /usr/share/fonts && mkdir chinese
chmod -R 755 /usr/share/fonts/chinese
cd chinese/ && rz simhei.ttf
yum -y install ttmkfdir
ttmkfdir -e /usr/share/X11/fonts/encodings/encodings.dir
vim /etc/fonts/fonts.conf
<dir>/usr/share/fonts</dir>
<dir>/usr/share/fonts/chinese</dir>
<dir>/usr/share/X11/fonts/Type1</dir> <dir>/usr/share/X11/fonts/TTF</dir> <dir>/usr/local/share/fonts</dir>
<dir prefix="xdg">fonts</dir>
<!-- the following element will be removed in the future -->
<dir>~/.fonts</dir>

fc-cache
fc-list :lang=zh
```

# RocketMQ

``` 
mkdir /data2/RocketMQ
vim docker-compose.yml
version: '3.5'
services:
  rmqnamesrv:
    image: foxiswho/rocketmq:server
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - ./data/logs:/opt/logs
      - ./data/store:/opt/store
    networks:
        rmq:
          aliases:
            - rmqnamesrv




  rmqbroker:
    image: foxiswho/rocketmq:broker
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
    volumes:
      - ./data/logs:/opt/logs
      - ./data/store:/opt/store
      - ./data/brokerconf/broker.conf:/etc/rocketmq/broker.conf
    environment:
        NAMESRV_ADDR: "rmqnamesrv:9876"
        JAVA_OPTS: " -Duser.home=/opt"
        JAVA_OPT_EXT: "-server -Xms128m -Xmx128m -Xmn128m"
    command: mqbroker -c /etc/rocketmq/broker.conf
    depends_on:
      - rmqnamesrv
    networks:
      rmq:
        aliases:
          - rmqbroker




  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    ports:
      - 8088:8080
    environment:
        JAVA_OPTS: "-Drocketmq.namesrv.addr=rmqnamesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false"
    depends_on:
      - rmqnamesrv
    networks:
      rmq:
        aliases:
          - rmqconsole




networks:
  rmq:
    name: rmq
    driver: bridge


vim /data2/RocketMQ/data/brokerconf/broker.conf
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
brokerIP1=192.168.2.7
defaultTopicQueueNums=4
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
listenPort=10911
deleteWhen=04
fileReservedTime=120
mapedFileSizeCommitLog=1073741824
mapedFileSizeConsumeQueue=300000
diskMaxUsedSpaceRatio=88
maxMessageSize=65536
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH


docker-compose up -d
docker-compose down
http://192.168.2.7:8088/
```

# jenkins

``` 
rpm -qa|grep jenkins    搜索已安装
rpm -e --nodeps jenkins-2.83-1.1.noarch
find / -name jenkins*|xargs rm -rf
rpm -ivh jdk-8u171-linux-x64.rpm        安装java
http://mirrors.jenkins.io/redhat/jenkins-2.180-1.1.noarch.rpm
rpm -ivh jenkins-2.180-1.1.noarch.rpm       安装jenkins新版本
vi /etc/sysconfig/jenkins
JENKINS_USER="root"
JENKINS_PORT="181"
vim /etc/rc.d/init.d/jenkins 修改candidates
/data2/jdk/bin/java
systemctl daemon-reload
systemctl restart jenkins
http://192.168.2.7:181
cat /var/lib/jenkins/secrets/initialAdminPassword    初始密码串并安装默认的插件
vim /var/lib/jenkins/hudson.model.UpdateCenter.xml    必须在填写完密码后修改
改https为http或者改为http://mirror.xmission.com/jenkins/updates/update-center.json
systemctl restart jenkins
若页面空白/var/lib/jenkins/config.xml
<authorizationStrategy class="hudson.security.AuthorizationStrategy$Unsecured"/>
<securityRealm class="hudson.security.SecurityRealm$None"/>

 
系统管理-全局工具配置:/data2/maven    /data2/jdk  /data2/maven/conf/settings.xml
安装插件：Maven Integration，GitHub plugin，Git plugin
新建任务时，丢弃旧的构建，保持构建的天数3，保持构建的最大个数5
```
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1564814498470.png)
## 定时删除none的docker镜像

手动执行：docker rmi $(docker images -f "dangling=true" -q)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565602173041.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565602184220.png)
定时构建语法：
``` 
每天凌晨2:00跑一次 
H 2  * * *

每隔5分钟构建一次
H/5 * * * *

每两小时构建一次
H H/2 * * *

每天中午12点定时构建一次
H 12 * * *   或0 12 * * *（0这种写法也被H替代了）

每天下午18点前定时构建一次
H 18 * * *
 
每15分钟构建一次
H/15 * * * *   或*/5 * * * *(这种方式已经被第一种替代了，jenkins也不推荐这种写法了)
 
周六到周日，18点-23点，三小时构建一次
H 18-23/3 * * 6-7 
```
shell脚本

``` 
echo ---------------Clear-Images...------------------
clearImagesList=$(docker images -f "dangling=true" -q)
if [ ! -n "$clearImagesList" ]; then
echo "no images need  clean up."
else
docker rmi $(docker images -f "dangling=true" -q)
echo "clear success."
fi
```

# cron
   

``` lsl
              每隔5秒执行一次：*/5 * * * * ?

                 每隔1分钟执行一次：0 */1 * * * ?

                 每天23点执行一次：0 0 23 * * ?

                 每天凌晨1点执行一次：0 0 1 * * ?

                 每月1号凌晨1点执行一次：0 0 1 1 * ?

                 每月最后一天23点执行一次：0 0 23 L * ?

                 每周星期天凌晨1点实行一次：0 0 1 ? * L

                 在26分、29分、33分执行一次：0 26,29,33 * * * ?

                 每天的0点、13点、18点、21点都执行一次：0 0 0,13,18,21 * * ?
```
# top
top -d 2 -c -p 123456 //每隔2秒显示pid是12345的进程的资源使用情况，并显式该进程启动的命令行参数
``` 
M —根据驻留内存大小进行排序 
P —根据CPU使用百分比大小进行排序 
T —根据时间/累计时间进行排序 
c —切换显示命令名称和完整命令行 
t —切换显示进程和CPU信息 
m —切换显示内存信息 
l —切换显示平均负载和启动时间信息 
o —改变显示项目的顺序 
f —从当前显示中添加或删除项目 
S —切换到累计模式 
s —改变两次刷新之间的延迟时间。系统将提示用户输入新的时间，单位为s。如果有小数，就换算成ms。
q —退出top程序 
i —忽略闲置和僵尸进程。这是一个开关式的命令 
k —终止一个进程
```
更改显示内容通过 f 键可以选择显示的内容。按 f 键之后会显示列的列表，按 a-z 即可显示或隐藏对应的列，最后按回车键确定。 
## Example

``` tap
top - 01:06:48 up  1:22,  1 user,  load average: 0.06, 0.60, 0.48
Tasks:  29 total,   1 running,  28 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.3% us,  1.0% sy,  0.0% ni, 98.7% id,  0.0% wa,  0.0% hi,  0.0% si
Mem:    191272k total,   173656k used,    17616k free,    22052k buffers
Swap:   192772k total,        0k used,   192772k free,   123988k cached

PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
root      16   0  7976 2456 1980 S  0.7  1.3   0:11.03 sshd
root      16   0  2128  980  796 R  0.7  0.5   0:02.72 top
root      16   0  1992  632  544 S  0.0  0.3   0:00.90 init
root      34  19     0    0    0 S  0.0  0.0   0:00.00 ksoftirqd/0
root      RT   0     0    0    0 S  0.0  0.0   0:00.00 watchdog/0
```
统计信息区前五行是系统整体的统计信息。第一行是任务队列信息，同 uptime 命令的执行结果。其内容如下：

``` css
01:06:48    当前时间
up 1:22    系统运行时间，格式为时:分
1 user    当前登录用户数
load average: 0.06, 0.60, 0.48    系统负载，即任务队列的平均长度。三个数值分别为 1分钟、5分钟、15分钟前到现在的平均值。
```
第二、三行为进程和CPU的信息。当有多个CPU时，这些内容可能会超过两行。内容如下：

``` 
total 进程总数
running 正在运行的进程数
sleeping 睡眠的进程数
stopped 停止的进程数
zombie 僵尸进程数
Cpu(s): 
0.3% us 用户空间占用CPU百分比
1.0% sy 内核空间占用CPU百分比
0.0% ni 用户进程空间内改变过优先级的进程占用CPU百分比
98.7% id 空闲CPU百分比
0.0% wa 等待输入输出的CPU时间百分比
0.0%hi：硬件CPU中断占用百分比
0.0%si：软中断占用百分比
0.0%st：虚拟机占用百分比
```
最后两行为内存信息。内容如下：

``` 
Mem:
191272k total    物理内存总量
173656k used    使用的物理内存总量
17616k free    空闲内存总量
22052k buffers    用作内核缓存的内存量
Swap: 
192772k total    交换区总量
0k used    使用的交换区总量
192772k free    空闲交换区总量
123988k cached    缓冲的交换区总量,内存中的内容被换出到交换区，而后又被换入到内存，但使用过的交换区尚未被覆盖，该数值即为这些内容已存在于内存中的交换区的大小,相应的内存再次被换出时可不必再对交换区写入。
```
进程信息区统计信息区域的下方显示了各个进程的详细信息。首先来认识一下各列的含义

``` 
序号  列名    含义
a    PID     进程id
b    PPID    父进程id
c    RUSER   Real user name
d    UID     进程所有者的用户id
e    USER    进程所有者的用户名
f    GROUP   进程所有者的组名
g    TTY     启动进程的终端名。不是从终端启动的进程则显示为 ?
h    PR      优先级
i    NI      nice值。负值表示高优先级，正值表示低优先级
j    P       最后使用的CPU，仅在多CPU环境下有意义
k    %CPU    上次更新到现在的CPU时间占用百分比
l    TIME    进程使用的CPU时间总计，单位秒
m    TIME+   进程使用的CPU时间总计，单位1/100秒
n    %MEM    进程使用的物理内存百分比
o    VIRT    进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
p    SWAP    进程使用的虚拟内存中，被换出的大小，单位kb。
q    RES     进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
r    CODE    可执行代码占用的物理内存大小，单位kb
s    DATA    可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
t    SHR     共享内存大小，单位kb
u    nFLT    页面错误次数
v    nDRT    最后一次写入到现在，被修改过的页面数。
w    S       进程状态(D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程)
x    COMMAND 命令名/命令行
y    WCHAN   若该进程在睡眠，则显示睡眠中的系统函数名
z    Flags   任务标志，参考 sched.h
```

> VIRT：virtual memory usage。Virtual这个词很神，一般解释是：virtual adj.虚的, 实质的,[物]有效的, 事实上的。到底是虚的还是实的？让Google给Define之后，将就明白一点，就是这东西还是非物质的，但是有效果的，不发生在真实世界的，发生在软件世界的等等。这个内存使用就是一个应用占有的地址空间，只是要应用程序要求的，就全算在这里，而不管它真的用了没有。写程序怕出错，又不在乎占用的时候，多开点内存也是很正常的。
> RES：resident memory usage。常驻内存。这个值就是该应用程序真的使用的内存，但还有两个小问题，一是有些东西可能放在交换盘上了（SWAP），二是有些内存可能是共享的。
> SHR：shared memory。共享内存。就是说这一块内存空间有可能也被其他应用程序使用着；而Virt － Shr似乎就是这个程序所要求的并且没有共享的内存空间。
> DATA：数据占用的内存。如果top没有显示，按f键可以显示出来。这一块是真正的该程序要求的数据空间，是真正在运行中要使用的。
> SHR是一个潜在的可能会被共享的数字，如果只开一个程序，也没有别人共同使用它；VIRT里面的可能性更多，比如它可能计算了被许多X的库所共享的内存；RES应该是比较准确的，但不含有交换出去的空间；但基本可以说RES是程序当前使用的内存量。

# Q1:-bash: fork: Cannot allocate memory
进程数满了,echo 1000000 > /proc/sys/kernel/pid_max,echo "kernel.pid_max=1000000 " >> /etc/sysctl.conf,sysctl -p
top:展示进程视图，监控服务器进程数值默认进入top时，各进程是按照CPU的占用量来排序的,-f查看实际内存占用量

详情见：
https://github.com/OneJane/blog