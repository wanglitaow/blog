@[TOC]

http://192.168.2.5:181/view/all/newJob 构建一个maven项目ht-micro-record-service-note-provider
添加jenkins主机公钥到gitlab，并生成全局凭据
1.Username with password root/123456
2.SSH Username with private key Enter Directly,添加gitlab服务器私钥

parent.relativePath修改为<relativePath />，发布单个服务时设定一个空值将始终从仓库中获取，不从本地路径获取
# 分布式构建

# 基础服务

``` xml
<build>
    <finalName>ht-micro-record-commons-domain</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <classifier>exec</classifier>  <!--被依赖的包加该配置-->
                <mainClass>com.ht.micro.record.commons.CommonsMapperApplication</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
```
# 微服务提供者

``` xml
	<distributionManagement>
		<repository>
			<id>nexus-releases</id>
			<name>Nexus Release Repository</name>
			<url>http://192.168.2.7:182/repository/maven-releases/</url>
		</repository>
		<snapshotRepository>
			<id>nexus-snapshots</id>
			<name>Nexus Snapshot Repository</name>
			<url>http://192.168.2.7:182/repository/maven-snapshots/</url>
		</snapshotRepository>
	</distributionManagement>
<build>
    <finalName>ht-micro-record-service-note-consumer</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
			    <!--被其他服务引用必须增加该配置-->
			    <classifier>exec</classifier>
                <mainClass>com.ht.micro.record.service.consumer.NoteConsumerServiceApplication</mainClass>
            </configuration>
        </plugin>
        <!-- docker的maven插件，官网 https://github.com/spotify/docker-maven-plugin -->
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>0.4.13</version>
            <configuration>
                <imageName>192.168.2.5:5000/${project.artifactId}:${project.version}</imageName>
                <baseImage>jdk1.8</baseImage>
                <entryPoint>["java", "-jar","/${project.build.finalName}.jar"]</entryPoint>
                <resources>
                    <resource>
                        <targetPath>/</targetPath>
                        <directory>${project.build.directory}</directory>
                        <include>${project.build.finalName}.jar</include>
                    </resource>
                </resources>
                <dockerHost>http://192.168.2.5:2375</dockerHost>
            </configuration>
        </plugin>
    </plugins>
</build>
```

# Jenkins
export app_name="ht-micro-record-service-note-consumer"
export app_version="1.0.0-SNAPSHOT"
sh /usr/local/docker/ht-micro-record/deploy.sh

``` bash

#!/bin/bash
# 判断项目是否在运行
DATE=`date +%s`
last_app_name=`echo "$app_name-expire-$DATE"`
if docker ps | grep $app_name;then
  docker stop $app_name
  docker rename $app_name $last_app_name
  docker run -di --name=$app_name -v /opt/template/:/opt/template/ -e JAVA_OPTS='-Xmx3g -Xms3g' --net=host 192.168.2.7:5000/$app_name:$app_version
# 判断项目是否存在
elif docker ps -a | grep $app_name;then
  docker start $app_name
else
  docker run -di --name=$app_name -v /opt/template/:/opt/template/ -e JAVA_OPTS='-Xmx3g -Xms3g' --net=host 192.168.2.7:5000/$app_name:$app_version
fi

```

vim /usr/local/bin/dokill

``` perl
docker images | grep -e $*|awk '{print $3}'|sed '2p'|xargs docker rmi
chmod a+x /usr/local/bin/dokill
dokill tensquare_recruit
```
Build时必须clean，项目名必须小写
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563789548337.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1564397581018.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563789561923.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563789568408.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1563789572978.png)

Q1:Failed to execute goal on project  : Could not resolve dependencies for

> 对最父级项目clean install，再最子项目clean install

Q2: repackage failed: Unable to find main class -> [Help 1]

> 构建显示缺少主类

``` 
@SpringBootApplication
public class CommonsApplication {
    public static void main(String[] args) {


    }
}
```



详情见：
https://github.com/OneJane/blog