@[TOC]
# 限流
## sentinel存储
 

``` xml
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
```
application.yml

``` css
spring:
  application:
    name: ht-micro-record-service-user

  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
    sentinel:
      transport:
        port: 8719
        dashboard: 192.168.2.7:19
```
访问ht-micro-record-service-user服务，http://localhost:9506/apply/page/1/2
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565244022641.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565244045582.png)
快速的调用两次http://localhost:9506/apply/page/1/2 接口之后，第三次调用被限流了

## nacos存储
  

``` xml
      <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
```
application.yml

``` 
spring:
  cloud:
    sentinel:
      datasource:
        ds:
          nacos:
            dataId: ${spring.application.name}-sentinel
            groupId: DEFAULT_GROUP
            rule-type: flow
            server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
```
nacos中配置ht-micro-record-service-user-sentinel
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565245358260.png)

``` json
[
        {
        "resource": "/apply/page/1/2",
        "limitApp": "default",
        "grade": 1,
        "count": 5,
        "strategy": 0,
        "controlBehavior": 0,
        "clusterMode": false
        }
]
```
进入http://192.168.2.7:190/#/dashboard/identity/ht-micro-record-service-user 访问
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565245759454.png)

* Sentinel控制台中修改规则：仅存在于服务的内存中，不会修改Nacos中的配置值，重启后恢复原来的值。
* Nacos控制台中修改规则：服务的内存中规则会更新，Nacos中持久化规则也会更新，重启后依然保持。
## 限流捕获处理异常
ht-micro-record-service-dubbo-provider

``` xml
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
```
com.ht.micro.record.service.dubbo.provider.DubboProviderApplication

``` aspectj
// 注解支持的配置Bean
    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
```

bootstrap.yml

``` 
spring:
  application:
    name: ht-micro-record-service-dubbo-provider
  cloud:
    nacos:
      config:
        file-extension: yaml
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
    sentinel:
      transport:
        port: 8720
        dashboard: 192.168.2.7:190
      datasource:
        ds:
          nacos:
            dataId: ${spring.application.name}-sentinel
            groupId: DEFAULT_GROUP
            rule-type: flow
            data-type: json
            server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
```
nacos配置ht-micro-record-service-dubbo-provider-sentinel

``` json
[
  {
    "resource": "protected-resource",
    "controlBehavior": 2,
    "count": 1,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  }
]
```
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565252128686.png)
nacos配置ht-micro-record-service-dubbo-provider.yaml

``` css
spring:
  application:
    name: ht-micro-record-service-dubbo-provider
  
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850

server:
  port: 9502
```

com.ht.micro.record.service.dubbo.provider.controller.ProviderController

``` 
@GetMapping("/port")
    @SentinelResource(value = "protected-resource", blockHandler = "handleBlock")
    public Object port() {
        return "port= "+ port + ", name=" + applicationContext.getEnvironment().getProperty("user.name");
    }
    public String  handleBlock(BlockException ex) {
        return "限流了";
    }
```
启动服务后http://192.168.2.7:190/#/dashboard/flow/ht-micro-record-service-dubbo-provider 
快速访问http://localhost:9502/provider/port 每秒超过1个请求将会显示限流了
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565252299866.png)
# 熔断
   

``` kotlin
 @GetMapping("name")
    @SentinelResource(value = "getName", fallback = "getNameFallback")
    public String userName(String name){
        for (int i = 0; i < 100000000L; i++) {
        }
        return "getName " + name;
    }

    // 该方法降级处理函数，参数要与原函数getName相同，并且返回值类型也要与原函数相同，此外，该方法必须与原函数在同一个类中
    public String getNameFallback(String name){
        return "getNameFallback";
    }
```
访问http://localhost:9502/provider/name
查看http://192.168.2.7:190/#/dashboard/identity/ht-micro-record-service-dubbo-provider
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565254838577.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565254852481.png)
根据响应时间，大于10毫秒，则熔断降级（要连续超过5个请求超过10毫秒才会熔断） 
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565254921781.png)
快速访问http://localhost:9502/provider/name

``` typescript
@GetMapping("name")
    @SentinelResource(value = "getName", fallback = "getNameFallback")
    public String userName(String name){
        for (int i = 0; i < 100000000L; i++) {
            throw new RuntimeException();
        }
        return "getName " + name;
    }

    // 该方法降级处理函数，参数要与原函数getName相同，并且返回值类型也要与原函数相同，此外，该方法必须与原函数在同一个类中
    public String getNameFallback(String name){
        return "getNameFallback";
    }
```

![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565255315318.png)
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565255392581.png)
显示抛出是10个异常，然后返回结果是熔断处理方法返回结果
详情见：
https://github.com/OneJane/blog