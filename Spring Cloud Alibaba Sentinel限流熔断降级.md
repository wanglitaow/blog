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

# 流控规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565256038192.png)
# 降级规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565256058092.png)
timeWindow为熔断恢复时间
熔断模式，当熔断触发后，需要等待timewindow时间，再关闭熔断器。
- 0 根据rt时间，当超过指定规则的时间连续超过5笔，则触发熔断。
- 1 根据异常比例熔断 DEGRADE_GRADE_EXCEPTION_RATIO
- 2 根据单位时间内异常总数做熔断
# 热点规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565256075475.png)
# 系统规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565256088028.png)
# 授权规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565256102707.png)

# Sentinel整合Nacos动态发布

``` crmsh
git clone https://github.com/alibaba/Sentinel.git
cd Sentinel/sentinel-dashboard
```
vim pom.xml
   

``` xml
     <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
            <!--<scope>test</scope>-->
        </dependency>
```
修改resources/app/scripts/directives/sidebar/sidebar.html

``` dust
<li ui-sref-active="active">
    <a ui-sref="dashboard.flowV1({app: entry.app})">
        <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则
    </a>
</li>
```
为

``` dust
<li ui-sref-active="active">
    <a ui-sref="dashboard.flow({app: entry.app})">
        <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则
    </a>
</li>
```

com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfigConstant

``` processing
public final class NacosConfigConstant {

    public static final String GROUP_ID = "DEFAULT_GROUP";

    public static final String FLOW_DATA_ID_POSTFIX = "-flow-rules";
    public static final String PARAM_FLOW_DATA_ID_POSTFIX = "-param-flow-rules";
    public static final String DEGRADE_DATA_ID_POSTFIX = "-degrade-rules";
    public static final String SYSTEM_DATA_ID_POSTFIX = "-system-rules";
    public static final String AUTHORITY_DATA_ID_POSTFIX = "-authority-rules";

}
```
com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfigProperties

``` typescript
@Component
public class NacosConfigProperties {

    @Value("${houyi.nacos.server.ip}")
    private String ip;

    @Value("${houyi.nacos.server.port}")
    private String port;

    @Value("${houyi.nacos.server.namespace}")
    private String namespace;

    @Value("${houyi.nacos.server.group-id}")
    private String groupId;

    public String getIp() {
        return ip;
    }
    public void setIp(String ip) {
        this.ip = ip;
    }
    public String getPort() {
        return port;
    }
    public void setPort(String port) {
        this.port = port;
    }
    public String getNamespace() {
        return namespace;
    }
    public void setNamespace(String namespace) {
        this.namespace = namespace;
    }
    public String getGroupId() {
        return groupId;
    }
    public void setGroupId(String groupId) {
        this.groupId = groupId;
    }
    public String getServerAddr() {
        return this.getIp()+":"+this.getPort();
    }
    @Override
    public String toString() {
        return "NacosConfigProperties [ip=" + ip + ", port=" + port + ", namespace="
                + namespace + ", groupId=" + groupId + "]";
    }
}
```
com.alibaba.csp.sentinel.dashboard.rule.nacos.NacosConfig

``` aspectj
@Configuration
public class NacosConfig {

    @Autowired
    private NacosConfigProperties nacosConfigProperties;

    @Bean
    public ConfigService nacosConfigService() throws Exception {
        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, nacosConfigProperties.getServerAddr());
        if(nacosConfigProperties.getNamespace() != null && !"".equals(nacosConfigProperties.getNamespace()))
            properties.put(PropertyKeyConst.NAMESPACE, nacosConfigProperties.getNamespace());
        return ConfigFactory.createConfigService(properties);
    }
}
```
application.properties

``` stylus
houyi.nacos.server.ip=192.168.2.7
houyi.nacos.server.port=8848
houyi.nacos.server.namespace=
houyi.nacos.server.group-id=DEFAULT_GROUP
```
## FlowRule
com.alibaba.csp.sentinel.dashboard.rule.nacos.FlowRuleNacosPublisher
``` groovy
@Component("flowRuleNacosPublisher")
public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>> {

    @Autowired
    private ConfigService configService;

    @Autowired
    private NacosConfigProperties nacosConfigProperties;

    @Override
    public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigConstant.FLOW_DATA_ID_POSTFIX,
                nacosConfigProperties.getGroupId(),
                JSON.toJSONString(rules.stream().map(FlowRuleEntity::toRule).collect(Collectors.toList())));
    }
}
```
com.alibaba.csp.sentinel.dashboard.rule.nacos.FlowRuleNacosProvider

``` groovy
@Component("flowRuleNacosProvider")
public class FlowRuleNacosProvider implements DynamicRuleProvider<List<FlowRuleEntity>> {

    private static Logger logger = LoggerFactory.getLogger(FlowRuleNacosProvider.class);
    @Autowired
    private ConfigService configService;

    @Autowired
    private NacosConfigProperties nacosConfigProperties;

    @Override
    public List<FlowRuleEntity> getRules(String appName) throws Exception {
        String rulesStr = configService.getConfig(appName + NacosConfigConstant.FLOW_DATA_ID_POSTFIX,
                nacosConfigProperties.getGroupId(), 3000);
        logger.info("nacosConfigProperties{}:", nacosConfigProperties);
        logger.info("从Nacos中获取到限流规则信息{}", rulesStr);
        if (StringUtil.isEmpty(rulesStr)) {
            return new ArrayList<>();
        }
        List<FlowRule> rules = RuleUtils.parseFlowRule(rulesStr);

        if (rules != null) {
            return rules.stream().map(rule -> FlowRuleEntity.fromFlowRule(appName, nacosConfigProperties.getIp(), Integer.valueOf(nacosConfigProperties.getPort()), rule))
                    .collect(Collectors.toList());
        } else {
            return new ArrayList<>();
        }
    }
}
```
com.alibaba.csp.sentinel.dashboard.controller.FlowControllerV1
将原始HTTP调用改为对应的Provider及Publisher
去除
 

``` aspectj
@Autowired
    private SentinelApiClient sentinelApiClient;
```
新增Qualifier和Component值保持一致
  

``` less
@Autowired
    @Qualifier("flowRuleNacosProvider")
    private DynamicRuleProvider<List<FlowRuleEntity>> provider;
    @Autowired
    @Qualifier("flowRuleNacosPublisher")
    private DynamicRulePublisher<List<FlowRuleEntity>> publisher;
@GetMapping("/rules")
    public Result<List<FlowRuleEntity>> apiQueryMachineRules(HttpServletRequest request,
                                                             @RequestParam String app,
                                                             @RequestParam String ip,
                                                             @RequestParam Integer port) {
        AuthUser authUser = authService.getAuthUser(request);
        authUser.authTarget(app, PrivilegeType.READ_RULE);

        if (StringUtil.isEmpty(app)) {
            return Result.ofFail(-1, "app can't be null or empty");
        }
        if (StringUtil.isEmpty(ip)) {
            return Result.ofFail(-1, "ip can't be null or empty");
        }
        if (port == null) {
            return Result.ofFail(-1, "port can't be null");
        }
        try {
            List<FlowRuleEntity> rules = provider.getRules(app);
            rules = repository.saveAll(rules);
            return Result.ofSuccess(rules);
        } catch (Throwable throwable) {
            logger.error("Error when querying flow rules", throwable);
            return Result.ofThrowable(-1, throwable);
        }
    }	
    private boolean publishRules(String app, String ip, Integer port) {
        List<FlowRuleEntity> rules = repository.findAllByMachine(MachineInfo.of(app, ip, port));
        try {
            publisher.publish(app, rules);
            logger.info("添加限流规则成功{}", JSON.toJSONString(rules.stream().map(FlowRuleEntity::toRule).collect(Collectors.toList())));
            return true;
        } catch (Exception e) {
            logger.info("添加限流规则失败{}",JSON.toJSONString(rules.stream().map(FlowRuleEntity::toRule).collect(Collectors.toList())));
            e.printStackTrace();
            return false;
        }
    }	
```
## DegradeRule
com.alibaba.csp.sentinel.dashboard.rule.nacos.DegradeRuleNacosPublisher

``` groovy
@Component("degradeRuleNacosPublisher")
public class DegradeRuleNacosPublisher implements DynamicRulePublisher<List<DegradeRuleEntity>> {

    @Autowired
    private ConfigService configService;

    @Autowired
    private NacosConfigProperties nacosConfigProperties;

    @Override
    public void publish(String app, List<DegradeRuleEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigConstant.DEGRADE_DATA_ID_POSTFIX,
                nacosConfigProperties.getGroupId(),
                JSON.toJSONString(rules.stream().map(DegradeRuleEntity::toRule).collect(Collectors.toList())));
    }
}
```
com.alibaba.csp.sentinel.dashboard.rule.nacos.DegradeRuleNacosProvider

``` groovy
@Component("degradeRuleNacosProvider")
public class DegradeRuleNacosProvider implements DynamicRuleProvider<List<DegradeRuleEntity>> {

    private static Logger logger = LoggerFactory.getLogger(DegradeRuleNacosProvider.class);
    @Autowired
    private ConfigService configService;

    @Autowired
    private NacosConfigProperties nacosConfigProperties;

    @Override
    public List<DegradeRuleEntity> getRules(String appName) throws Exception {
        String rulesStr = configService.getConfig(appName + NacosConfigConstant.DEGRADE_DATA_ID_POSTFIX,
                nacosConfigProperties.getGroupId(), 3000);
        logger.info("nacosConfigProperties{}:", nacosConfigProperties);
        logger.info("从Nacos中获取到熔断降级规则信息{}", rulesStr);
        if (StringUtil.isEmpty(rulesStr)) {
            return new ArrayList<>();
        }
        List<DegradeRule> rules = RuleUtils.parseDegradeRule(rulesStr);

        if (rules != null) {
            return rules.stream().map(rule -> DegradeRuleEntity.fromDegradeRule(appName, nacosConfigProperties.getIp(), Integer.valueOf(nacosConfigProperties.getPort()), rule))
                    .collect(Collectors.toList());
        } else {
            return new ArrayList<>();
        }
    }
}
```
com.alibaba.csp.sentinel.dashboard.controller.DegradeController
去除

``` 
@Autowired
    private SentinelApiClient sentinelApiClient;
```
新增

``` 
@Autowired
    @Qualifier("degradeRuleNacosProvider")
    private DynamicRuleProvider<List<DegradeRuleEntity>> provider;

    @Autowired
    @Qualifier("degradeRuleNacosPublisher")
    private DynamicRulePublisher<List<DegradeRuleEntity>> publisher;
	@ResponseBody
    @RequestMapping("/rules.json")
    public Result<List<DegradeRuleEntity>> queryMachineRules(HttpServletRequest request, String app, String ip, Integer port) {
        AuthUser authUser = authService.getAuthUser(request);
        authUser.authTarget(app, PrivilegeType.READ_RULE);

        if (StringUtil.isEmpty(app)) {
            return Result.ofFail(-1, "app can't be null or empty");
        }
        if (StringUtil.isEmpty(ip)) {
            return Result.ofFail(-1, "ip can't be null or empty");
        }
        if (port == null) {
            return Result.ofFail(-1, "port can't be null");
        }
        try {
            List<DegradeRuleEntity> rules = provider.getRules(app);
            rules = repository.saveAll(rules);
            return Result.ofSuccess(rules);
        } catch (Throwable throwable) {
            logger.error("queryApps error:", throwable);
            return Result.ofThrowable(-1, throwable);
        }
    }
	   private boolean publishRules(String app, String ip, Integer port) {
        List<DegradeRuleEntity> rules = repository.findAllByMachine(MachineInfo.of(app, ip, port));
        try {
            publisher.publish(app, rules);
            logger.info("添加熔断降级规则成功{}", JSON.toJSONString(rules.stream().map(DegradeRuleEntity::toRule).collect(Collectors.toList())));
            return true;
        } catch (Exception e) {
            logger.info("添加熔断降级规则失败{}",JSON.toJSONString(rules.stream().map(DegradeRuleEntity::toRule).collect(Collectors.toList())));
            e.printStackTrace();
            return false;
        }
    }
```
重新打包启动

``` 
mvn clean package -DskipTests
cd sentinel-dashboard/target/
nohup java -Dserver.port=190 -Dcsp.sentinel.dashboard.server=localhost:190 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar &
```
项目中bootstrap.yml

``` yaml
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
        dashboard: localhost:8080
      datasource:
        ds:
          nacos:
            dataId: ${spring.application.name}-flow-rules
            groupId: DEFAULT_GROUP
            rule-type: flow # 流控
#            rule-type: degrade # 熔断
            data-type: json
            server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
        ds1:
          nacos:
            dataId: ${spring.application.name}-degrade-rules
            groupId: DEFAULT_GROUP
            rule-type: degrade # 熔断
            data-type: json
            server-addr: 192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
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

server:
  port: 9502
```
http://localhost:8080/#/dashboard 新增规则
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565313768651.png)
在http://192.168.2.7:8848/nacos 中自动同步
![enter description here](https://www.github.com/OneJane/blog/raw/master/小书匠/1565313831198.png)
暂时实现flow，degrade，
详情见：
https://github.com/OneJane/blog
