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