@[TOC]

1. 一个GroupId（项目）下面可以有很多个ArtifactId（模块），每个ArtifactId（模块）会有很多个Version（版本），每个Version（版本）一般被Packaging（打包）为jar、war、pom中的一种。
2. 父模块的packaging必须为pom,packaging的默认值为jar
3. module“组织”子模块,基于父模块pom的子模块相对目录名，不是子模块的artifactId
4. 多模块deploy的时候，根据子模块的相互依赖关系整理一个build顺序，然后依次build，独立模块之间根据配置顺序build
5. 子pom 会直接继承 父pom <properties /> 中声明的属性
6. 父pom 中仅仅使用dependencies 中dependency 依赖的包，会被 子pom 直接继承（不需要显式依赖）
7.  父pom 中仅仅使用plugins 中plugin依赖的插件，会被 子pom 直接继承（不需要显式依赖）
8.  父pom中可以使用dependecyManagement和pluginManagement来统一管理jar包和插件pugin，不会被子pom 直接继承。子pom如果希望继承该包或插件，则需要显式依赖，同时像 <version /> <url /> 等配置项可以不被显式写出，默认从父pom继承,同pluginManagement
9.  mvn 命令对应着maven项目生命周期，顺序为compile、test、package、install、depoly，执行其中任一周期，都会把前面周期 统一执行
10.  <dependency /> 具有依赖传递性，而<plugin />不具有依赖传递性
11.  安装jar包到本地

``` crystal
mvn install:install-file -DgroupId=org.csource.fastdfs -DartifactId=fastdfs -Dversion=1.2 -Dpackaging=jar 
-Dfile=D:\Project\pinyougou\fastDFSdemo\src\main\webapp\WEB-INF\lib\fastdfs_client_v1.20.jar
```
# properties

``` dust
<properties>
  <spring.version>2.5</spring.version>
</properties>

<version>${spring.version}</version>
```

# pluginManagement
父

``` xml
<pluginManagement>
    <plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.8.0</version>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
				<encoding>UTF-8</encoding>
			</configuration>
		</plugin>
	</plugins>
</pluginManagement>
```

子

``` xml
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
    </plugin>
</plugins>
```
由于 Maven 内置了 maven-compiler-plugin 与生命周期的绑定，因此子模块就不再需要任何 maven-compiler-plugin 的配置了。
# dependencyManagement

``` xml
<dependencyManagement>
  <dependencies>
      <dependency>
        <groupId>com.juvenxu.sample</groupId>
        <artifactid>sample-dependency-infrastructure</artifactId>
        <version>1.0-SNAPSHOT</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
  </dependencies>
</dependencyManagement>
```
## 优势
- Parent的dependencyManagement节点的作用是管理整个项目所需要用到的第三方依赖。只要约定了第三方依赖的坐标（GroupId:ArtifactId:version），后代模块即可通过GroupId:ArtifactId进行依赖的引入。其它元素如 version 和 scope 都能通过继承父 POM 的 dependencyManagement 得到这样能够避免依赖版本的冲突。当然，这里只是进行约定，并不会真正地引用依赖。
- 依赖统一管理(parent中定义，需要变动dependency版本，只要修改一处即可)；
- 代码简洁(子model只需要指定groupId、artifactId即可)
- dependencyManagement只会影响现有依赖的配置，但不会引入依赖，即子model不会继承parent中dependencyManagement所有预定义的depandency，只引入需要的依赖即可，简单说就是“按需引入依赖”或者“按需继承”；因此，在parent中严禁直接使用depandencys预定义依赖，坏处是子model会自动继承depandencys中所有预定义依赖；
## 劣势
单继承：maven的继承跟java一样，单继承，也就是说子model中只能出现一个parent标签；parent模块中，dependencyManagement中预定义太多的依赖，造成pom文件过长，而且很乱；
		      通过父pom的parent继承的方法，只能继承一个parent。实际开发中，用户很可能需要继承自己公司的标准parent配置，这个时候可以使用 scope=import 来实现多继承。
> 解决：scope=import只能用在dependencyManagement里面,且仅用于type=pom的dependency,要继承多个，可以在dependencyManagement通过非继承的方式来引入这段依赖管理配置，添加依赖scope=import，type=pom

``` 
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>1.3.3.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
		 <dependency>
            <groupId>com.test.sample</groupId>
            <artifactid>base-parent1</artifactId>
            <version>1.0.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
 
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```
自己的项目里面就不需要继承SpringBoot的module了，而可以继承自己项目的module了。
# Scope
- provided 被依赖项目理论上可以参与编译、测试、运行等阶段，相当于compile，但是再打包阶段做了exclude的动作。,servlet-api和jsp-api都是由tomcat等servlet容器负责提供的包,这个 jar 包已由应用服务器提供,这里引入只用于开发,不进行打包
- runtime被依赖项目无需参与项目的编译，但是会参与到项目的测试和运行,在编译的时候我们不需要 JDBC API 的 jar 包，而在运行的时候我们才需要 JDBC 驱动包。
- compile（默认）被依赖项目需要参与到当前项目的编译，测试，打包，运行等阶段。打包的时候通常会包含被依赖项目。
- test 被依赖项目仅仅参与测试相关的工作，包括测试代码的编译，执行,Junit 测试。
- system 被依赖项不会从 maven 仓库中查找，而是从本地系统中获取，systemPath 元素用于制定本地系统中 jar 文件的路径
# 发布

``` 
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
mvn deploy发布到私服,在项目 pom.xml 中设置的版本号添加 SNAPSHOT 标识的都会发布为 SNAPSHOT 版本，没有 SNAPSHOT 标识的都会发布为 RELEASE 版本
```
Nexus 3.0 不支持页面上传，可使用 maven 命令：

``` haml
mvn deploy:deploy-file
  -DgroupId=com.github.axet
  -DartifactId=kaptcha
  -Dversion=0.0.9
  -Dpackaging=jar
  -Dfile=E:\kaptcha-0.0.9.jar
  -Durl=http://192.168.2.5:182/repository/maven-releases/
  -DrepositoryId=nexus-releases
```

要求jar的pom中的repository.id和settings.xml中一致

详情见：
https://github.com/OneJane/blog