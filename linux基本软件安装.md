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
git clone git@101.101.101.101:/usr/local/git/learngit.git
```