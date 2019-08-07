@[TOC]
# ht-micro-record-service-job
``` dust
    <parent>
        <groupId>com.htdc</groupId>
        <artifactId>ht-micro-record-dependencies</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../ht-micro-record-dependencies/pom.xml</relativePath>
    </parent>

    <artifactId>ht-micro-record-service-job</artifactId>
    <packaging>jar</packaging>

    <name>ht-micro-record-service-job</name>
    <url>http://www.htdatacloud.com/</url>
    <inceptionYear>2019-Now</inceptionYear>

    <dependencies>
        <dependency>
            <groupId>com.xuxueli</groupId>
            <artifactId>xxl-job-core</artifactId>
            <version>2.1.0</version>
        </dependency>

        <!-- Spring Boot Begin -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
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
                    <groupId>com.google.guava</groupId>
                    <artifactId>guava</artifactId>
                </exclusion>
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
            <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>com.alibaba</groupId>
                    <artifactId>fastjson</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- Spring Cloud End -->

        <!-- Projects Begin -->
        <dependency>
            <groupId>com.htdc</groupId>
            <artifactId>ht-micro-record-commons-service</artifactId>
            <version>${project.parent.version}</version>
        </dependency>
        <!-- Projects End -->
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>ht.micro.record.service.job.JobServiceApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
bootstrap.properties

``` stylus
spring.application.name=ht-micro-record-service-xxl-job-config
spring.cloud.nacos.config.file-extension=yaml
spring.cloud.nacos.config.server-addr=192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
```
application.yaml

``` 
spring:
  datasource:
    druid:
      url: jdbc:mysql://192.168.2.7:185/ht_micro_record?useUnicode=true&characterEncoding=utf-8&useSSL=false
      username: root
      password: 123456
      initial-size: 1
      min-idle: 1
      max-active: 20
      test-on-borrow: true
      driver-class-name: com.mysql.jdbc.Driver
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
      config:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850

server:
  port: 9701

xxl:
  job:
    executor:
      logpath: logs/xxl-job/jobhandler
      appname: xxl-job-executor
      port: 1234
      logretentiondays: -1
      ip: 192.168.3.233
    admin:
      addresses: http://192.168.2.7:183/xxl-job-admin
    accessToken:
```
nacos配置

``` less
spring:
  application:
    name: ht-micro-record-service-xxl-job

  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
    sentinel:
      transport:
        port: 8719
        dashboard: 192.168.2.7:190


server:
  port: 9701

management:
  endpoints:
    web:
      exposure:
        include: "*"
```
com/ht/micro/record/service/job/JobServiceApplication.java

``` less
@SpringBootApplication(scanBasePackages = "com.ht.micro.record")
@EnableDiscoveryClient
@MapperScan(basePackages = "com.ht.micro.record.commons.mapper")
public class JobServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(JobServiceApplication.class, args);
    }

}
```
com/ht/micro/record/service/job/config/XxlJobConfig.java

``` kotlin
@Configuration
@ComponentScan(basePackages = "com.ht.micro.record.service.job.handler")
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.executor.appname}")
    private String appName;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;


    @Bean(initMethod = "start", destroyMethod = "destroy")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppName(appName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }

}
```
com/ht/micro/record/service/job/handler/TestJobHandler.java

``` scala
@JobHandler(value="testJobHandler")
@Component
public class TestJobHandler extends IJobHandler {


    @Override
    public ReturnT<String> execute(String param) throws Exception {
        XxlJobLogger.log("XXL-JOB, Hello World.");

        for (int i = 0; i < 5; i++) {
            XxlJobLogger.log("beat at:" + i);
            TimeUnit.SECONDS.sleep(2);
        }

        return SUCCESS;
    }
}
```
http://192.168.2.7:183/xxl-job-admin

详情见：
https://github.com/OneJane/blog