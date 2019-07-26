@[TOC]

# 配置
application.yml

``` 
server:
  port: 10205
spring:
  datasource:
      base:
        #监控统计拦截的filters
        filters: stat
        type: com.alibaba.druid.pool.DruidDataSource
        jdbc-url: jdbc:mysql://192.168.1.250:3306/htbl_test?useUnicode=true&characterEncoding=UTF-8
        username: root
        password: 123456
        driver-class-name: com.mysql.jdbc.Driver
        max-idle: 10
        max-wait: 10000
        min-idle: 5
        initial-size: 5
        validation-query: SELECT 1
        test-on-borrow: false
        test-while-idle: true
        time-between-eviction-runs-millis: 18800
      dic:
        #监控统计拦截的filters
        filters: stat
        type: com.alibaba.druid.pool.DruidDataSource
        jdbc-url: jdbc:mysql://192.168.1.48:3306/ht_nlp?useUnicode=true&characterEncoding=UTF-8&useSSL=false
        username: root
        password: Sonar@1234
        driver-class-name: com.mysql.jdbc.Driver
        max-idle: 10
        max-wait: 10000
        min-idle: 5
        initial-size: 5
        validation-query: SELECT 1
        test-on-borrow: false
        test-while-idle: true
        time-between-eviction-runs-millis: 18800
```
# 数据源1配置
BaseMybatisConfig

``` 
@Configuration
@MapperScan(basePackages = {"com.ht.micro.record.commons.mapper.baseMapper"}, sqlSessionTemplateRef = "baseSqlSessionTemplate")
public class BaseMybatisConfig {

    @Value("${spring.datasource.base.filters}")
    String filters;
    @Value("${spring.datasource.base.driver-class-name}")
    String driverClassName;
    @Value("${spring.datasource.base.username}")
    String username;
    @Value("${spring.datasource.base.password}")
    String password;
    @Value("${spring.datasource.base.jdbc-url}")
    String url;


    @Bean(name="baseDataSource")
    @Primary//必须加此注解，不然报错，下一个类则不需要添加 spring.datasource
    @ConfigurationProperties(prefix="spring.datasource.base")//prefix值必须是application.properteis中对应属性的前缀
    public DataSource baseDataSource() throws SQLException {


        DruidDataSource druid = new DruidDataSource();
        // 监控统计拦截的filters
        druid.setFilters(filters);
        // 配置基本属性
        druid.setDriverClassName(driverClassName);
        druid.setUsername(username);
        druid.setPassword(password);
        druid.setUrl(url);

        return druid;
    }


    @Bean(name="baseSqlSessionFactory")
    @Primary
    public SqlSessionFactory baseSqlSessionFactory(@Qualifier("baseDataSource")DataSource dataSource)throws Exception{
        // 创建Mybatis的连接会话工厂实例
        SqlSessionFactoryBean bean=new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);//// 设置数据源bean
        //添加XML目录
        ResourcePatternResolver resolver=new PathMatchingResourcePatternResolver();
        try{
            bean.setMapperLocations(resolver.getResources("classpath:baseMapper/*Mapper.xml"));//// 设置mapper文件路径
            return bean.getObject();
        }catch(Exception e){
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }

    @Bean(name="baseSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate baseSqlSessionTemplate(@Qualifier("baseSqlSessionFactory")SqlSessionFactory sqlSessionFactory)throws Exception{
        SqlSessionTemplate template=new SqlSessionTemplate(sqlSessionFactory);//使用上面配置的Factory
        return  template;
    }

}

```
# 数据源2配置

``` 
@Configuration
@MapperScan(basePackages = {"com.ht.micro.record.commons.mapper.dicMapper"}, sqlSessionTemplateRef = "dicSqlSessionTemplate")
public class DicMybatisConfig {
    @Value("${spring.datasource.dic.filters}")
    String filters;
    @Value("${spring.datasource.dic.driver-class-name}")
    String driverClassName;
    @Value("${spring.datasource.dic.username}")
    String username;
    @Value("${spring.datasource.dic.password}")
    String password;
    @Value("${spring.datasource.dic.jdbc-url}")
    String url;


    @Bean(name = "dicDataSource")
    @ConfigurationProperties(prefix="spring.datasource.dic")
    public DataSource dicDataSource() throws SQLException {

        DruidDataSource druid = new DruidDataSource();
        // 监控统计拦截的filters
       // druid.setFilters(filters);
        // 配置基本属性
        druid.setDriverClassName(driverClassName);
        druid.setUsername(username);
        druid.setPassword(password);
        druid.setUrl(url);

       /* //初始化时建立物理连接的个数
        druid.setInitialSize(initialSize);
        //最大连接池数量
        druid.setMaxActive(maxActive);
        //最小连接池数量
        druid.setMinIdle(minIdle);
        //获取连接时最大等待时间，单位毫秒。
        druid.setMaxWait(maxWait);
        //间隔多久进行一次检测，检测需要关闭的空闲连接
        druid.setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
        //一个连接在池中最小生存的时间
        druid.setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
        //用来检测连接是否有效的sql
        druid.setValidationQuery(validationQuery);
        //建议配置为true，不影响性能，并且保证安全性。
        druid.setTestWhileIdle(testWhileIdle);
        //申请连接时执行validationQuery检测连接是否有效
        druid.setTestOnBorrow(testOnBorrow);
        druid.setTestOnReturn(testOnReturn);
        //是否缓存preparedStatement，也就是PSCache，oracle设为true，mysql设为false。分库分表较多推荐设置为false
        druid.setPoolPreparedStatements(poolPreparedStatements);
        // 打开PSCache时，指定每个连接上PSCache的大小
        druid.setMaxPoolPreparedStatementPerConnectionSize(maxPoolPreparedStatementPerConnectionSize);
*/
        return druid;
    }

    @Bean(name = "dicSqlSessionFactory")
    public SqlSessionFactory dicSqlSessionFactory(@Qualifier("dicDataSource")DataSource dataSource)throws Exception{
        SqlSessionFactoryBean bean=new SqlSessionFactoryBean();
        bean.setDataSource(dataSource);
        //添加XML目录
        ResourcePatternResolver resolver=new PathMatchingResourcePatternResolver();
        try{
            bean.setMapperLocations(resolver.getResources("classpath:dicMapper/*Mapper.xml"));
            return bean.getObject();
        }catch(Exception e){
            e.printStackTrace();
            throw new RuntimeException(e);
        }
    }

    @Bean(name="dicSqlSessionTemplate")
    public SqlSessionTemplate dicSqlSessionTemplate(@Qualifier("dicSqlSessionFactory")SqlSessionFactory sqlSessionFactory)throws Exception{
        SqlSessionTemplate template=new SqlSessionTemplate(sqlSessionFactory);//使用上面配置的Factory
        return  template;
    }

}
```
# 启动类
NoteProviderServiceApplication

``` 
@SpringBootApplication(scanBasePackages = "com.ht.micro.record")
@EnableDiscoveryClient
@MapperScan(basePackages = "com.ht.micro.record.commons.mapper")
@EnableSwagger2
public class NoteProviderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(NoteProviderServiceApplication.class, args);
    }
}
```
# 测试
## 服务层
WebDicController

``` 
@RestController
@RequestMapping(value = "webdic")
public class WebDicController extends AbstractBaseController<TbUser> {

    @Autowired
    private WebDicService webDicService;
 
    @ApiOperation(value = "获取所有字典")
    @RequestMapping
    public List<WebDic> getAll() {
        return webDicService.getAll();
    }

}
```
TApplyController

``` 
@RestController
@RequestMapping(value = "apply")
public class TApplyController extends AbstractBaseController {
    @Autowired
    private TApplyService tApplyService;


    /** http://localhost:10105/apply/asked/任大龙
     * @param name
     * @return
     */
    @ApiImplicitParams({
            @ApiImplicitParam(name = "askedName", value = "被询问人名", required = true, paramType = "path")
    })
    @GetMapping(value = "asked/{name}")
    public List<TApply> getByAskedName(@PathVariable String name){
        return tApplyService.getByAskedName(name);
    }
}
```
WebDicService

``` 
@Service
public class WebDicService {
    @Autowired
    WebDicMapper webDicMapper;

    public List<WebDic> getAll() {
        return webDicMapper.selectAll();
    }
}
```
TApplyService

``` 
public interface TApplyService extends BaseCrudService<TApply> {
    List<TApply> getByAskedName(String name);
}
```
TApplyServiceImpl

``` 
@Service
public class TApplyServiceImpl extends BaseCrudServiceImpl<TApply, TApplyMapper> implements TApplyService {

    @Autowired
    private TApplyMapper tApplyMapper;

    public List<TApply> getByAskedName(String name) {
        return tApplyMapper.selectByAskedName(name);
    }

}
```
## mapper层
com.ht.micro.record.commons.mapper.baseMapper.TApplyMapper

``` 
public interface TApplyMapper extends MyMapper<TApply> {

    /**
     * 根据名称查找
     * @param name
     * @return
     */
    List<TApply> selectByAskedName(String name);
}
```

com.ht.micro.record.commons.mapper.dicMapper.WebDicMapper

``` 
public interface WebDicMapper extends MyMapper<WebDic> {
}
```
tk.mybatis.mapper.MyMapper

``` 
public interface MyMapper<T> extends Mapper<T>, MySqlMapper<T> {
}
```
baseMapper/TApplyMapper.xml

``` 
<mapper namespace="com.ht.micro.record.commons.mapper.baseMapper.TApplyMapper">
  <resultMap id="BaseResultMap" type="com.ht.micro.record.commons.domain.TApply">
    <!--
      WARNING - @mbg.generated
    -->
    <id column="id" jdbcType="INTEGER" property="id" />
    <result column="applyer_police_num" jdbcType="VARCHAR" property="applyerPoliceNum" />
    <result column="applyer_name" jdbcType="VARCHAR" property="applyerName" />
    <result column="asked_name" jdbcType="VARCHAR" property="askedName" />
    <result column="applyer_id" jdbcType="INTEGER" property="applyerId" />
    <result column="unit_id" jdbcType="INTEGER" property="unitId" />
    <result column="case_info_ids" jdbcType="VARCHAR" property="caseInfoIds" />
    <result column="case_count" jdbcType="INTEGER" property="caseCount" />
    <result column="apply_time" jdbcType="TIMESTAMP" property="applyTime" />
    <result column="apply_state" jdbcType="VARCHAR" property="applyState" />
    <result column="apply_goal" jdbcType="VARCHAR" property="applyGoal" />
    <result column="approve_time" jdbcType="TIMESTAMP" property="approveTime" />
    <result column="approve_state" jdbcType="VARCHAR" property="approveState" />
    <result column="approver_id" jdbcType="INTEGER" property="approverId" />
    <result column="approve_prop" jdbcType="VARCHAR" property="approveProp" />
  </resultMap>

  <select id="selectByAskedName" parameterType="string" resultMap="BaseResultMap">
    select *
    from t_apply
    where asked_name = #{name,jdbcType=VARCHAR}
  </select>
</mapper>
```
dicMapper/WebDicMapper.xml

``` 
<mapper namespace="com.ht.micro.record.commons.mapper.dicMapper">
  <resultMap id="BaseResultMap" type="com.ht.micro.record.commons.domain.WebDic">
    <!--
      WARNING - @mbg.generated
    -->
    <id column="id" jdbcType="INTEGER" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="type" jdbcType="VARCHAR" property="type" />
  </resultMap>
</mapper>
```

详情见：
https://github.com/OneJane/blog