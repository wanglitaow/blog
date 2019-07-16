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
