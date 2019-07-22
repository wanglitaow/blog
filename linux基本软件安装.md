@[TOC](Linux常用软件安装)
# jdk+maven

``` tar zxf jdk-8u60-linux-x64.tar.gz -C /usr/local/
tar zxf apache-maven-3.6.1-bin.tar.gz -C /usr/local/
vim /etc/profile
JAVA_HOME=/usr/local/jdk1.8.0_60
JRE_HOME=$JAVA_HOME/jre
MAVEN_HOME=/usr/local/apache-maven-3.6.1
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$MAVEN_HOME/bin
CLASSPATH=:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib/dt.jar
export JAVA_HOME JRE_HOME PATH CLASSPATH MAVEN_HOME
```
# Nexus

``` vim /usr/local/docker/nexus/docker-compose.yml
version: '3.1'
services:
  nexus:
    restart: always
    image: sonatype/nexus3
    container_name: nexus
    ports:
      - 182:8081
    volumes:
      - /usr/local/docker/nexus/data:/nexus-data
docker-compose up -d
cat data/admin.password        查看密码，账户为admin
```
http://192.168.2.5:182/ admin 123456
在maven的settings.xml中配置Nexus认证节点

``` <settings
        xmlns="http://maven.apache.org/SETTINGS/1.0.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <!-- localRepository
   | The path to the local repository maven will use to store artifacts.
   |
   | Default: ${user.home}/.m2/repository
  <localRepository>/path/to/local/repo</localRepository>
  -->
    <localRepository>E:\apache-maven-3.6.1\respository</localRepository>
    <servers>  
        <server>  
          <id>nexus-releases</id>  
          <username>admin</username>  
          <password>123456</password>  
        </server>  
        <server>  
          <id>nexus-snapshots</id>  
          <username>admin</username>  
          <password>123456</password>  
        </server>  
    </servers>  
  
  <mirrors>   
    <mirror>   
      <id>nexus-releases</id>   
      <mirrorOf>maven-releases</mirrorOf>   
      <url>http://192.168.2.5:182/repository/maven-releases/</url>   
    </mirror>  
    <mirror>   
      <id>nexus-snapshots</id>   
      <mirrorOf>maven-snapshots</mirrorOf>   
      <url>http://192.168.2.5:182/repository/maven-snapshots/</url>   
    </mirror>    
    <mirror>
        <id>nexus-aliyun</id>
        <mirrorOf>central</mirrorOf>
        <name>Nexus aliyun</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public</url>
    </mirror>
  </mirrors>   
   
  <profiles>  
   <profile>  
      <id>nexus</id>  
      <repositories>  
        <repository>  
          <id>nexus-releases</id>  
          <url>http://192.168.2.5:182/repository/maven-releases/</url>  
          <releases><enabled>true</enabled></releases>  
          <snapshots><enabled>true</enabled></snapshots>  
        </repository>  
        <repository>  
          <id>nexus-snapshots</id>  
          <url>http://192.168.2.5:182/repository/maven-snapshots/</url>  
          <releases><enabled>true</enabled></releases>  
          <snapshots><enabled>true</enabled></snapshots>  
        </repository>  
        <repository>
            <id>nexus-aliyun</id>
            <name>Nexus aliyun</name>
            <layout>default</layout>
            <url>http://maven.aliyun.com/nexus/content/groups/public</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
            <releases>
                <enabled>true</enabled>
            </releases>
        </repository>
      </repositories>  
      <pluginRepositories>  
         <pluginRepository>  
                <id>nexus-releases</id>  
                 <url>http://192.168.2.5:182/repository/maven-releases/</url>  
                 <releases><enabled>true</enabled></releases>  
                 <snapshots><enabled>true</enabled></snapshots>  
               </pluginRepository>  
               <pluginRepository>  
                 <id>nexus-snapshots</id>  
                  <url>http://192.168.2.5:182/repository/maven-snapshots/</url>  
                <releases><enabled>true</enabled></releases>  
                 <snapshots><enabled>true</enabled></snapshots>  
             </pluginRepository>  
             <pluginRepository>
                <id>aliyun</id>
                <name>aliyun</name>
                <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
                <releases>
                    <enabled>true</enabled>
                </releases>
                <snapshots>
                    <enabled>false</enabled>
                </snapshots>
                </pluginRepository>
         </pluginRepositories>  
    </profile>  
  </profiles>  
  <activeProfiles>  
      <activeProfile>nexus</activeProfile>  
  </activeProfiles>  
</settings>
```
> 地址在http://192.168.2.5:182/#admin/repository/repositories中查看URL idea配置maven,并开启always update snapshots

## 配置自动化部署

``` xml
<version>1.0.0-SNAPSHOT</version>
<distributionManagement>
   <repository>
      <id>nexus-releases</id>        <!-- ID 名称必须要与 settings.xml 中 Servers 配置的 ID 名称保持一致。-->
      <name>Nexus Release Repository</name>
      <url>http://192.168.2.5:182/repository/maven-releases/</url>
   </repository>
   <snapshotRepository>
      <id>nexus-snapshots</id>
      <name>Nexus Snapshot Repository</name>
      <url>http://192.168.2.5:182/repository/maven-snapshots/</url>
   </snapshotRepository>
</distributionManagement>
```
> mvn deploy发布到私服,在项目 pom.xml 中设置的版本号添加 SNAPSHOT 标识的都会发布为 SNAPSHOT 版本，没有 SNAPSHOT 标识的都会发布为 RELEASE 版本

## 上传第三方jar
Nexus 3.0 不支持页面上传，可使用 maven 命令：
``` mvn deploy:deploy-file
  -DgroupId=com.github.axet
  -DartifactId=kaptcha
  -Dversion=0.0.9
  -Dpackaging=jar
  -Dfile=E:\kaptcha-0.0.9.jar
  -Durl=http://192.168.2.5:182/repository/maven-releases/
  -DrepositoryId=nexus-releases
```
> 要求jar的pom中的repository.id和settings.xml中一致

# GitLab
## 方案1

``` docker run \
    --publish 1443:443 --publish 180:80 --publish 122:22 \
    --name gitlab \
    --volume /usr/local/docker/gitlab/config:/etc/gitlab \
    --volume /usr/local/docker/gitlab/logs:/var/log/gitlab \
    --volume /usr/local/docker/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce
```
## 方案2
vim /usr/local/docker/gitlab/docker-compose.yml
``` version: '3'
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

``` yum –y install git
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

``` git config --global http.sslVerify false
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

``` openssl x509 -req -days 3650 -in "/etc/gitlab/ssl/gitlab.example.com.csr" -signkey "/etc/gitlab/ssl/gitlab.example.com.key" -out "/etc/gitlab/ssl/gitlab.example.com.crt"
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
# RocketMQ安装

``` mkdir /usr/local/docker/RocketMQ
vim docker-compose.yml
version: '3.5'
services:
  rmqnamesrv:
    image: foxiswho/rocketmq:server
    container_name: rmqnamesrv
    ports:
      - 187:9876
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

vim /usr/local/docker/RocketMQ/data/brokerconf/broker.conf
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
brokerIP1=192.168.2.5
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
```

# Jenkins
## 方案0

``` dts
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
/usr/local/jdk1.8.0_60/bin/java
systemctl daemon-reload
systemctl restart jenkins
http://192.168.2.5:181
cat /var/lib/jenkins/secrets/initialAdminPassword    初始密码串并安装默认的插件
vim /var/lib/jenkins/hudson.model.UpdateCenter.xml    必须在填写完密码后修改
改https为http或者改为http://mirror.xmission.com/jenkins/updates/update-center.json
systemctl restart jenkins
若页面空白/var/lib/jenkins/config.xml
<authorizationStrategy class="hudson.security.AuthorizationStrategy$Unsecured"/>
<securityRealm class="hudson.security.SecurityRealm$None"/>



tar zxvf apache-maven-3.3.9-bin.tar.gz
mv apache-maven-3.3.9 /usr/local/maven
vim /usr/local/maven/conf/settings.xml
<localRepository>/usr/local/repository</localRepository>
mkdir /usr/local/repository

系统管理-全局工具配置:/usr/local/apache-maven-3.6.1    /usr/java/jdk1.8.0_171-amd64  /usr/local/apache-maven-3.6.1/conf/settings.xml
安装插件：Maven Integration plugin，GitHub plugin，Git plugin
新建任务时，丢弃旧的构建，保持构建的天数3，保持构建的最大个数5
```
## 方案1

``` docker run -d -p 181:8080 -p 50000:50000 -v jenkins:/var/jenkins_home -v /etc/localtime:/etc/localtime --name jenkins docker.io/jenkins/jenkins  
```
## 方案2

``` mkdir /usr/local/docker/jenkins -p
vim docker-compose.yml
jenkins:
    container_name: jenkins
    image: jenkinsci/jenkins:2.14
    ports:
        - "181:8080"
        - "50000:50000"
    environment:
        - JAVA_OPTS=-Duser.timezone=Asia/Shanghai
    volumes:
        - $PWD/jenkins_home:/var/jenkins_home
    restart: always
mkdir -p jenkins_home
chown 1000:1000 jenkins_home
docker-compose up -d
```
## 基本使用

``` http://192.168.2.5:181/login
docker exec jenkins tail /var/jenkins_home/secrets/initialAdminPassword    
首次启动太慢 /var/jenkins_home/hudson.model.UpdateCenter.xml更改地址http://mirror.xmission.com/jenkins/updates/update-center.json


docker cp apache-maven-3.6.1-bin.tar.gz jenkins:/    将宿主机文件拷到jenkins容器
docker cp jenkins:/opt/apache-maven-3.6.1-bin.tar.gz /  将容器文件拷贝到宿主机
docker exec -it -u root jenkins bash
tar zxf apache-maven-3.6.1-bin.tar.gz -C /usr/local/
cd /usr/local/ && mv apache-maven-3.6.1/ maven
mv /etc/apt/sources.list sources.list.bak
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse">>/etc/apt/sources.list
echo "deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse">>/etc/apt/sources.list
apt-get update
apt-get install gnupg -y --allow-unauthenticated
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3B4FE6ACC0B21F32
apt-get update
apt-get install vim lrzsz -y
vim /etc/profile                        vim ~/.bashrc
MAVEN_HOME=/usr/local/maven
PATH=$PATH:$MAVEN_HOME/bin
export PATH MAVEN_HOME
source /etc/profile                        source ~/.bashrc

全局配置jdk git maven 默认settings配置文件路径
/usr/local/openjdk-8                            /usr/lib/jvm/java-8-openjdk-amd64
/usr/bin/git
/usr/local/maven
/usr/local/maven/conf/settings.xml

ssh-keygen -t rsa -C "15806204096@qq.com"
docker exec -u root jenkins tail ~/.ssh/id_rsa.pub    获取公钥
```


详情见：
https://github.com/OneJane/blog
https://www.jianshu.com/u/b2a63c970be4
https://www.cnblogs.com/codewj/