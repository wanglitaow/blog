@[TOC]
# 基础服务通用类

# 集成Swagger2
ht-micro-record-commons/pom.xml

``` 
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
</dependency>
```
com.ht.micro.record.commons.config.Swagger2Configuration

``` 
@Configuration
public class Swagger2Configuration {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.ht.micro.record"))  // 配置controller地址
                .paths(PathSelectors.any())
                .build();
    }


    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("微服务 API 文档")
                .description("微服务 API 网关接口，http://www.htdatacloud.com/")
                .termsOfServiceUrl("http://www.htdatacloud.com")
                .version("1.0.0")
                .build();
    }
}
```
com.ht.micro.record.UserServiceApplication

``` stylus
@EnableSwagger2

```
com.ht.micro.record.user.controller.UserController

``` less
@ApiOperation(value = "用户注册", notes = "参数为实体类，注意用户名和邮箱不要重复")
@PostMapping
```
# 服务提供消费

配置项目skywalking链路追踪并启动

``` 
-javaagent:E:\Project\ht-micro-record\ht-micro-record-external-skywalking\agent\skywalking-agent.jar
-Dskywalking.agent.service_name=ht-micro-record-service-user
-Dskywalking.collector.backend_service=192.168.2.7:11800
```
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1564815890566.png)
http://192.168.3.233:9501/user/10 调用服务
http://192.168.2.5:193/#/monitor/dashboard 查看skywalking的链路追踪 

详情见：
https://github.com/OneJane/blog