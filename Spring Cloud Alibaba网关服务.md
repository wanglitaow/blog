@[TOC]
# ht-micro-record-service-gateway
``` dust
<parent>
        <groupId>com.htdc</groupId>
        <artifactId>ht-micro-record-dependencies</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../ht-micro-record-dependencies/pom.xml</relativePath>
    </parent>

    <artifactId>ht-micro-record-service-gateway</artifactId>
    <packaging>jar</packaging>

    <name>ht-micro-record-service-gateway</name>
    <url>http://www.htdatacloud.com/</url>
    <inceptionYear>2019-Now</inceptionYear>

    <dependencies>
        <!-- Spring Boot Begin -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!-- Spring Boot End -->

        <!-- Spring Cloud Begin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba</groupId>
                    <artifactId>fastjson</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.google.guava</groupId>
                    <artifactId>guava</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.google.guava</groupId>
                    <artifactId>guava</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.alibaba</groupId>
                    <artifactId>fastjson</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.google.code.findbugs</groupId>
                    <artifactId>jsr305</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.hdrhistogram</groupId>
                    <artifactId>HdrHistogram</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <!-- Spring Cloud End -->

        <!-- Commons Begin -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
        </dependency>
        <!-- Commons Begin -->
    </dependencies>

    <build>
        <plugins>
            <!-- docker的maven插件，官网 https://github.com/spotify/docker-maven-plugin -->
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.13</version>
                <configuration>
                    <imageName>192.168.2.7:5000/${project.artifactId}:${project.version}</imageName>
                    <baseImage>onejane-jdk1.8 </baseImage>
                    <entryPoint>["java", "-jar","/${project.build.finalName}.jar"]</entryPoint>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                    <dockerHost>http://192.168.2.7:2375</dockerHost>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.ht.micro.record.service.gateway.GatewayServiceApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
bootstrap.properties

``` stylus
spring.application.name=ht-micro-record-service-gateway-config
spring.main.allow-bean-definition-overriding=true
spring.cloud.nacos.config.file-extension=yaml
spring.cloud.nacos.config.server-addr=192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
```

nacos配置

``` yaml
spring:
  application:
    name: ht-micro-record-service-gateway
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
    sentinel:
      transport:
        port: 8721
        dashboard: 192.168.2.7:190
    gateway:
      discovery:
        locator:
          enabled: true
      routes:
         http://localhost:9000/api/user/user/20 访问user服务
        - id: HT-MICRO-RECORD-SERVICE-USER
          uri: lb://ht-micro-record-service-user
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=2



server:
  port: 9000


feign:
  sentinel:
    enabled: true


management:
  endpoints:
    web:
      exposure:
        include: "*"


logging:
  level:
    org.springframework.cloud.gateway: debug
```
GatewayServiceApplication.java

``` aspectj
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class GatewayServiceApplication {

    // ----------------------------- 解决跨域 Begin -----------------------------

    private static final String ALL = "*";
    private static final String MAX_AGE = "18000L";

    @Bean
    public RouteDefinitionLocator discoveryClientRouteDefinitionLocator(DiscoveryClient discoveryClient, DiscoveryLocatorProperties properties) {
        return new DiscoveryClientRouteDefinitionLocator(discoveryClient, properties);
    }

    @Bean
    public ServerCodecConfigurer serverCodecConfigurer() {
        return new DefaultServerCodecConfigurer();
    }

    @Bean
    public WebFilter corsFilter() {
        return (ServerWebExchange ctx, WebFilterChain chain) -> {
            ServerHttpRequest request = ctx.getRequest();
            if (!CorsUtils.isCorsRequest(request)) {
                return chain.filter(ctx);
            }
            HttpHeaders requestHeaders = request.getHeaders();
            ServerHttpResponse response = ctx.getResponse();
            HttpMethod requestMethod = requestHeaders.getAccessControlRequestMethod();
            HttpHeaders headers = response.getHeaders();
            headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, requestHeaders.getOrigin());
            headers.addAll(HttpHeaders.ACCESS_CONTROL_ALLOW_HEADERS, requestHeaders.getAccessControlRequestHeaders());
            if (requestMethod != null) {
                headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_METHODS, requestMethod.name());
            }
            headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
            headers.add(HttpHeaders.ACCESS_CONTROL_EXPOSE_HEADERS, ALL);
            headers.add(HttpHeaders.ACCESS_CONTROL_MAX_AGE, MAX_AGE);
            if (request.getMethod() == HttpMethod.OPTIONS) {
                response.setStatusCode(HttpStatus.OK);
                return Mono.empty();
            }
            return chain.filter(ctx);
        };
    }

    // ----------------------------- 解决跨域 End -----------------------------

    public static void main(String[] args) {
        SpringApplication.run(GatewayServiceApplication.class, args);
    }
}
```

详情见：
https://github.com/OneJane/blog