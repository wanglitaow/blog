@[TOC]
# ht-micro-record-service-dubbo
pom.xml
   

``` dts
 <modelVersion>4.0.0</modelVersion>
    <modules>
        <module>ht-micro-record-service-dubbo-provider</module>
        <module>ht-micro-record-service-dubbo-consumer</module>
    </modules>
    <parent>
        <groupId>com.htdc</groupId>
        <artifactId>ht-micro-record-dependencies</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath>../ht-micro-record-dependencies/pom.xml</relativePath>
    </parent>

    <artifactId>ht-micro-record-service-dubbo</artifactId>
    <packaging>pom</packaging>

    <name>ht-micro-record-service-dubbo</name>
    <url>http://www.htdatacloud.com/</url>
    <inceptionYear>2019-Now</inceptionYear>
```
# ht-micro-record-service-dubbo-api
 
pom.xml
``` xml
   <parent>
        <artifactId>ht-micro-record-service-dubbo</artifactId>
        <groupId>com.htdc</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <packaging>jar</packaging>

    <artifactId>ht-micro-record-service-dubbo-api</artifactId>
```
PortApi.java
``` java
public interface PortApi {
    String showPort();
}
```
# ht-micro-record-service-dubbo-provider
pom.xml
``` xml
<parent>
        <artifactId>ht-micro-record-service-dubbo</artifactId>
        <groupId>com.htdc</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>ht-micro-record-service-dubbo-provider</artifactId>

    <dependencies>
        <dependency>
            <groupId>com.htdc</groupId>
            <artifactId>ht-micro-record-service-dubbo-api</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator</artifactId>
        </dependency>
        <!-- Spring Cloud Begin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-dubbo</artifactId>
        </dependency>
        <!--elasticsearch-->
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>net.java.dev.jna</groupId>
            <artifactId>jna</artifactId>
            <!-- <version>3.0.9</version> -->
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>2.8.2</version>
        </dependency>
    </dependencies>


    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.ht.micro.record.service.dubbo.provider.DubboProviderApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
logback.xml 

``` dust
<configuration>

	<!-- 控制台输出 -->
	<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
		<layout class="ch.qos.logback.classic.PatternLayout">
			<!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符 -->
			<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
		</layout>
	</appender>

	<!--定义日志文件的存储地址 勿在 LogBack 的配置中使用相对路径 -->
	<property name="LOG_HOME" value="E:/log" />
	<!-- 按照每天生成日志文件 -->
	<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<!--日志文件输出的文件名 -->
			<FileNamePattern>${LOG_HOME}/log.%d{yyyy-MM-dd}.log</FileNamePattern>
			<MaxHistory>30</MaxHistory>
		</rollingPolicy>
		<layout class="ch.qos.logback.classic.PatternLayout">
			<pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
		</layout>
		<!--日志文件最大的大小 -->
		<triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
			<MaxFileSize>10MB</MaxFileSize>
		</triggeringPolicy>
	</appender>

	<!-- 日志输出级别 -->
	<root level="INFO">
		<appender-ref ref="STDOUT" />
		<appender-ref ref="FILE" />
	</root>

	<!--mybatis 日志配置-->
    <logger name="com.mapper" level="DEBUG" />

</configuration>
```
bootstrap.yml

``` yaml
spring:
  application:
    name: ht-micro-record-service-dubbo-provider
  cloud:
    nacos:
      config:
        file-extension: yaml
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
  main:
    allow-bean-definition-overriding: true

dubbo:
  scan:
    base-packages: com.ht.micro.record.service.dubbo.provider.dubbo
  protocol:
    name: dubbo
    port: -1
  registry:
    address: spring-cloud://192.168.2.7:8848?backup=192.168.2.7:8849,192.168.2.7:8850

es:
  nodes: 192.168.2.5:1800,192.168.2.7:1800
  host: 192.168.2.5,192.168.2.7
  port: 1801,1801
  types: doc
  clusterName: elasticsearch-cluster
  #笔录分析库
  blDbName: t_record_analyze
  #接警信息库
  jjDbName: p_answer_alarm
  #处警信息库
  cjDbName: p_handle_alarm
  #警情分析结果信息库
  jqfxDbName: t_alarm_analysis_result
  #标准化分析
  cjjxqDbName: t_cjjxq
  #接处警整合库
  jcjDbName: p_answer_handle_alarm
  #警情分析
  daDbName: t_data_analysis
  #警情结果表(neo4j使用)
  neo4jData: t_neo4j_data
  #市局接处警表
  cityDbName: city_answer_handle_alarm
```
nacos配置ht-micro-record-service-dubbo-provider.yaml

``` 
spring:
  application:
    name: ht-micro-record-service-dubbo-provider
  
  cloud:
    nacos:
      discovery:
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
    sentinel:
      transport:
        port: 8719
        dashboard: 192.168.2.7:190


server:
  port: 9502
```
com.ht.micro.record.service.dubbo.provider.DubboProviderApplication

``` less
@SpringBootApplication
@EnableDiscoveryClient
public class DubboProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboProviderApplication.class, args);
    }

}
```
com.ht.micro.record.service.dubbo.provider.utils.ElasticClient

``` 
@Service
public class ElasticClient {
    @Autowired
    private EsConfig esConfig;

    private static final Logger logger = LoggerFactory.getLogger(ElasticClient.class);
    
    private static BulkRequestBuilder bulkRequest;

    @Autowired
    private  ElasticClientSingleton elasticClientSingleton;
    @PostConstruct
    public void init() {

    	bulkRequest = elasticClientSingleton.getBulkRequest(esConfig);
    }



    

    /**
     * @param url
     * @param query
     * @return
     * @Description 发送请求
     * @Author 裴健(peij@htdatacloud.com)
     * @Date 2016年6月13日
     * @History
     * @his1
     */
    public static String postRequest(String url, String query) {
        RestTemplate restTemplate = new RestTemplate();
        MediaType type = MediaType.parseMediaType("application/json; charset=UTF-8");
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(type);
        headers.add("Accept", MediaType.APPLICATION_JSON.toString());
        HttpEntity<String> formEntity = new HttpEntity<String>(query, headers);
        String result = restTemplate.postForObject(url, formEntity, String.class);

        return result;
    }



    /**
     * action 提交操作
     */
//    public void action() {
//        int reqSize = bulkRequest.numberOfActions();
//        //读不到数据了，默认已经全部读取
//        if (reqSize == 0) {
//            bulkRequest.request().requests().clear();
//        }
//        bulkRequest.setTimeout(new TimeValue(1000 * 60 * 5)); //超时30秒
//        BulkResponse bulkResponse = bulkRequest.execute().actionGet();
//        //持久化异常
//        if (bulkResponse.hasFailures()) {
//            logger.error(bulkResponse.buildFailureMessage());
//            bulkRequest.request().requests().clear();
//        }
//        logger.info("import over...." + bulkResponse.getItems().length);
//    }


}
```
com.ht.micro.record.service.dubbo.provider.utils.ElasticClientSingleton

``` 
@Service
public class ElasticClientSingleton {
    protected final Logger logger = LoggerFactory.getLogger(ElasticClientSingleton.class);

    private AtomicInteger atomicPass = new AtomicInteger();    // 0 未初始化, 1 已初始化

    private TransportClient transportClient;
    private BulkRequestBuilder bulkRequest;


    public synchronized void init(EsConfig esConfig) {
        try {
            String ipArray = esConfig.getHost();
            String portArray = esConfig.getPort();
            String cluster = esConfig.getClusterName();

            Settings settings = Settings.builder()
                    .put("cluster.name", cluster)  //连接的集群名
                    .put("client.transport.ignore_cluster_name", true)
                    .put("client.transport.sniff", false)//如果集群名不对，也能连接
                    .build();
            transportClient = new PreBuiltTransportClient(settings);

            String[] ips = ipArray.split(",");
            String[] ports = portArray.split(",");
            for (int i = 0; i < ips.length; i++) {
                transportClient.addTransportAddress(new TransportAddress(InetAddress.getByName(ips[i]), Integer
                        .parseInt(ports[i])));
            }

            atomicPass.set(1);

        } catch (Exception e) {
            e.printStackTrace();
            logger.error(e.getMessage());

            atomicPass.set(0);
            destroy();
        }
    }

    public void destroy() {
        if (transportClient != null) {
            transportClient.close();
            transportClient = null;
        }
    }

    public BulkRequestBuilder getBulkRequest(EsConfig esConfig) {
        if (atomicPass.get() == 0) {    // 初始化
            init(esConfig);
        }

        bulkRequest = transportClient.prepareBulk();

        return bulkRequest;
    }

    public TransportClient getTransportClient(EsConfig esConfig) {
        if (atomicPass.get() == 0) {    // 初始化
            init(esConfig);
        }

        return transportClient;
    }
}
```
com.ht.micro.record.service.dubbo.provider.utils.EsConfig

``` 
@Service
public class EsConfig {

    @Value("${es.nodes}")
    private String nodes;

    @Value("${es.host}")
    private String host;

    @Value("${es.port}")
    private String port;

    @Value("${es.blDbName}")
    private String blDbName;

    @Value("${es.jjDbName}")
    private String jjDbName;

    @Value("${es.cjDbName}")
    private String cjDbName;

    @Value("${es.jqfxDbName}")
    private String jqfxDbName;

    @Value("${es.clusterName}")
    private String clusterName;

    @Value("${es.jjDbName}")
    private String answerDbName;

    @Value("${es.cjDbName}")
    private String handleDbName;

    @Value("${es.cjjxqDbName}")
    private String cjjxqDbName;

    @Value("${es.jcjDbName}")
    private String jcjDbName;

    @Value("${es.daDbName}")
    private String daDbName;

    @Value("${es.daDbName}")
    private String fxDbName;

    @Value("${es.types}")
    private String types;

    @Value("${es.neo4jData}")
    private String neo4jData;

    @Value("${es.cityDbName}")
    private String cityDbName;

    public String getCityDbName() {
        return cityDbName;
    }

    public String getTypes() {
        return types;
    }

    public String getFxDbName() {
        return fxDbName;
    }

    public String getDaDbName() {
        return daDbName;
    }

    public String getJcjDbName() {
        return jcjDbName;
    }

    public String getCjjxqDbName() {
        return cjjxqDbName;
    }

    public String getNodes() {
        return nodes;
    }

    public String getHost() {
        return host;
    }

    public String getPort() {
        return port;
    }

    public String getClusterName() {
        return clusterName;
    }

    public String getBlDbName() {
        return blDbName;
    }

    public String getJjDbName() {
        return jjDbName;
    }

    public String getCjDbName() {
        return cjDbName;
    }

    public String getJqfxDbName() {

        return jqfxDbName;
    }

    public String getNeo4jData() {
        return neo4jData;
    }
}

```
com.ht.micro.record.service.dubbo.provider.dubbo.PortApiImpl

``` 
@Service
public class PortApiImpl implements PortApi {

    @Value("${server.port}")
    private Integer port;

    @Override
    public String showPort() {
        return "port= "+ port;
    }

}
```
com.ht.micro.record.service.dubbo.provider.controller.ProviderController

``` 
@RestController
@RequestMapping("/provider")
public class ProviderController {

    @Autowired
    private ConfigurableApplicationContext applicationContext;

    @Value("${server.port}")
    private Integer port;

    @GetMapping("/port")
    public Object port() {
        return "port= "+ port + ", name=" + applicationContext.getEnvironment().getProperty("user.name");
    }

}
``` 

# ht-micro-record-service-dubbo-consumer
pom.xml

``` 
<parent>
        <artifactId>ht-micro-record-service-dubbo</artifactId>
        <groupId>com.htdc</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>ht-micro-record-service-dubbo-consumer</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.htdc</groupId>
            <artifactId>ht-micro-record-service-dubbo-api</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-dubbo</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
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
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.ht.micro.record.service.dubbo.consumer.DubboConsumerApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
```
bootstrap.yml 

``` 
spring:
  application:
    name: ht-micro-record-service-dubbo-consumer
  main:
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      config:
        file-extension: yaml
        server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850

# 与nacos兼容性不好，配置到项目中
dubbo:
  scan:
    base-packages: com.ht.micro.record.service.dubbo.consumer
  protocol:
    name: dubbo
    port: -1
  registry:
    address: spring-cloud://192.168.2.7:8848?backup=192.168.2.7:8849,192.168.2.7:8850
```
com.ht.micro.record.service.dubbo.consumer.DubboConsumerApplication

``` 
@SpringBootApplication(scanBasePackages = "com.ht.micro.record", exclude = {DataSourceAutoConfiguration.class})
@EnableFeignClients
@EnableDiscoveryClient
public class DubboConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboConsumerApplication.class, args);
    }

}

```
com.ht.micro.record.service.dubbo.consumer.service.PortService

``` 
@FeignClient(value = "ht-micro-record-service-dubbo-provider")
public interface PortService {
    @GetMapping(value = "/provider/port")
    String showPort();
}
```
com.ht.micro.record.service.dubbo.consumer.controller.ConsumerController

``` 
@RestController
@RequestMapping("/consumer")
public class ConsumerController {

    @Autowired
    private LoadBalancerClient loadBalancerClient;
    @Autowired
    private PortService portService;

    private RestTemplate restTemplate = new RestTemplate();

    @Reference(check = false)
    private PortApi portApi;

    @GetMapping("/rest")
    public Object rest() {
        ServiceInstance serviceInstance = loadBalancerClient.choose("ht-micro-record-service-dubbo-provider");
        String url = String.format("http://%s:%s/provider/port", serviceInstance.getHost(), serviceInstance.getPort());
        System.out.println("request url:" + url);
        return restTemplate.getForObject(url, String.class);
    }

    @GetMapping("/rpc")
    public Object rpc() {
        return portApi.showPort();
    }

    @GetMapping("/feign")
    public Object feign(){
        return portService.showPort();
    }


}
```

详情见：
https://github.com/OneJane/blog
