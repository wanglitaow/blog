@[TOC]
# 基础服务通用类
## com.ht.micro.record.commons
dto.AbstractBaseDomain
``` less
@Data
public abstract class AbstractBaseDomain implements Serializable {
    /**
     * 该注解需要保留，用于 tk.mybatis 回显 ID
     */
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;


    /**
     * 格式化日期，由于是北京时间（我们是在东八区），所以时区 +8
     */
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date created;
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date updated;
}
```
dto.AbstractBaseResult

``` scala
@Data
public abstract class AbstractBaseResult implements Serializable {

    /**
     * 此为内部类,JsonInclude.Include.NON_NULL去掉返回值中为null的属性
     */
    @Data
    @JsonInclude(JsonInclude.Include.NON_NULL)
    protected static class Links {
        private String self;
        private String next;
        private String last;
    }

    @Data
    @JsonInclude(JsonInclude.Include.NON_NULL)
    protected static class DataBean<T extends AbstractBaseDomain> {
        private String type;
        private Long id;
        private T attributes;
        private T relationships;
        private Links links;
    }
}
```
dto.BaseResultFactory

``` java
public class BaseResultFactory<T extends AbstractBaseDomain> {

    /**
     * 设置日志级别，用于限制发生错误时，是否显示调试信息(detail)
     *
     * @see ErrorResult#detail
     */
    public static final String LOGGER_LEVEL_DEBUG = "DEBUG";

    private static BaseResultFactory baseResultFactory;

    private BaseResultFactory() {

    }

    // 设置通用的响应
    private static HttpServletResponse response;

    public static BaseResultFactory getInstance(HttpServletResponse response) {
        if (baseResultFactory == null) {
            synchronized (BaseResultFactory.class) {
                if (baseResultFactory == null) {
                    baseResultFactory = new BaseResultFactory();
                }
            }
        }

        BaseResultFactory.response = response;
        // 设置通用响应
        baseResultFactory.initResponse();
        return baseResultFactory;
    }

    public static BaseResultFactory getInstance() {
        if (baseResultFactory == null) {
            synchronized (BaseResultFactory.class) {
                if (baseResultFactory == null) {
                    baseResultFactory = new BaseResultFactory();
                }
            }
        }

        return baseResultFactory;
    }



    /**
     * 构建单笔数据结果集
     *
     * @param self 当前请求路径
     * @return
     */
    public AbstractBaseResult build(String self, T attributes) {
        return new SuccessResult(self, attributes);
    }

    /**
     * 构建多笔数据结果集
     *
     * @param self 当前请求路径
     * @param next 下一页的页码
     * @param last 最后一页的页码
     * @return
     */
    public AbstractBaseResult build(String self, int next, int last, List<T> attributes) {
        return new SuccessResult(self, next, last, attributes);
    }

    /**
     * 构建请求错误的响应结构，调试显示detail，上线不显示，通过配置日志级别
     *
     * @param code   HTTP 状态码
     * @param title  错误信息
     * @param detail 调试信息
     * @param level  日志级别，只有 DEBUG 时才显示详情
     * @return
     */
    public AbstractBaseResult build(int code, String title, String detail, String level) {
        // 设置请求失败的响应码
        response.setStatus(code);

        if (LOGGER_LEVEL_DEBUG.equals(level)) {
            return new ErrorResult(code, title, detail);
        } else {
            return new ErrorResult(code, title, null);
        }
    }

    /**
     * 初始化 HttpServletResponse
     */
    private void initResponse() {
        // 需要符合 JSON API 规范
        response.setHeader("Content-Type", "application/vnd.api+json");
    }
}
```
dto.ErrorResult

``` less
@Data
@AllArgsConstructor
@EqualsAndHashCode(callSuper = false)
// JSON 不显示为 null 的属性
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResult extends AbstractBaseResult {
    private int code;
    private String title;

    /**
     * 调试信息
     */
    private String detail;
}
```
dto.SuccessResult

``` zephir
@Data
@EqualsAndHashCode(callSuper = false)
public class SuccessResult<T extends AbstractBaseDomain> extends AbstractBaseResult {
    private Links links;
    private List<DataBean> data;

    /**
     * 请求的结果（单笔）
     * @param self 当前请求路径
     * @param attributes 领域模型
     */
    public SuccessResult(String self, T attributes) {
        links = new Links();
        links.setSelf(self);

        createDataBean(null, attributes);
    }

    /**
     * 请求的结果（分页）
     * @param self 当前请求路径
     * @param next 下一页的页码
     * @param last 最后一页的页码
     * @param attributes 领域模型集合
     */
    public SuccessResult(String self, int next, int last, List<T> attributes) {
        links = new Links();
        links.setSelf(self);
        links.setNext(self + "?page=" + next);
        links.setLast(self + "?page=" + last);

        attributes.forEach(attribute -> createDataBean(self, attribute));
    }

    /**
     * 创建 DataBean
     * @param self 当前请求路径
     * @param attributes 领域模型
     */
    private void createDataBean(String self, T attributes) {
        if (data == null) {
            data = new ArrayList<>();
        }

        DataBean dataBean = new DataBean();
        dataBean.setId(attributes.getId());
        dataBean.setType(attributes.getClass().getSimpleName());
        dataBean.setAttributes(attributes);

        if (StringUtils.isNotBlank(self)) {
            Links links = new Links();
            links.setSelf(self + "/" + attributes.getId());
            dataBean.setLinks(links);
        }

        data.add(dataBean);
    }
}
```
utils.MapperUtils

``` php
public class MapperUtils {
    private final static ObjectMapper objectMapper = new ObjectMapper();

    public static ObjectMapper getInstance() {
        return objectMapper;
    }

    /**
     * 转换为 JSON 字符串
     *
     * @param obj
     * @return
     * @throws Exception
     */
    public static String obj2json(Object obj) throws Exception {
        return objectMapper.writeValueAsString(obj);
    }

    /**
     * 转换为 JSON 字符串，忽略空值
     *
     * @param obj
     * @return
     * @throws Exception
     */
    public static String obj2jsonIgnoreNull(Object obj) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        return mapper.writeValueAsString(obj);
    }

    /**
     * 转换为 JavaBean
     *
     * @param jsonString
     * @param clazz
     * @return
     * @throws Exception
     */
    public static <T> T json2pojo(String jsonString, Class<T> clazz) throws Exception {
        objectMapper.configure(DeserializationFeature.ACCEPT_SINGLE_VALUE_AS_ARRAY, true);
        return objectMapper.readValue(jsonString, clazz);
    }

    /**
     * 字符串转换为 Map<String, Object>
     *
     * @param jsonString
     * @return
     * @throws Exception
     */
    public static <T> Map<String, Object> json2map(String jsonString) throws Exception {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        return mapper.readValue(jsonString, Map.class);
    }

    /**
     * 字符串转换为 Map<String, T>
     */
    public static <T> Map<String, T> json2map(String jsonString, Class<T> clazz) throws Exception {
        Map<String, Map<String, Object>> map = objectMapper.readValue(jsonString, new TypeReference<Map<String, T>>() {
        });
        Map<String, T> result = new HashMap<String, T>();
        for (Map.Entry<String, Map<String, Object>> entry : map.entrySet()) {
            result.put(entry.getKey(), map2pojo(entry.getValue(), clazz));
        }
        return result;
    }

    /**
     * 深度转换 JSON 成 Map
     *
     * @param json
     * @return
     */
    public static Map<String, Object> json2mapDeeply(String json) throws Exception {
        return json2MapRecursion(json, objectMapper);
    }

    /**
     * 把 JSON 解析成 List，如果 List 内部的元素存在 jsonString，继续解析
     *
     * @param json
     * @param mapper 解析工具
     * @return
     * @throws Exception
     */
    private static List<Object> json2ListRecursion(String json, ObjectMapper mapper) throws Exception {
        if (json == null) {
            return null;
        }

        List<Object> list = mapper.readValue(json, List.class);

        for (Object obj : list) {
            if (obj != null && obj instanceof String) {
                String str = (String) obj;
                if (str.startsWith("[")) {
                    obj = json2ListRecursion(str, mapper);
                } else if (obj.toString().startsWith("{")) {
                    obj = json2MapRecursion(str, mapper);
                }
            }
        }

        return list;
    }

    /**
     * 把 JSON 解析成 Map，如果 Map 内部的 Value 存在 jsonString，继续解析
     *
     * @param json
     * @param mapper
     * @return
     * @throws Exception
     */
    private static Map<String, Object> json2MapRecursion(String json, ObjectMapper mapper) throws Exception {
        if (json == null) {
            return null;
        }

        Map<String, Object> map = mapper.readValue(json, Map.class);

        for (Map.Entry<String, Object> entry : map.entrySet()) {
            Object obj = entry.getValue();
            if (obj != null && obj instanceof String) {
                String str = ((String) obj);

                if (str.startsWith("[")) {
                    List<?> list = json2ListRecursion(str, mapper);
                    map.put(entry.getKey(), list);
                } else if (str.startsWith("{")) {
                    Map<String, Object> mapRecursion = json2MapRecursion(str, mapper);
                    map.put(entry.getKey(), mapRecursion);
                }
            }
        }

        return map;
    }

    /**
     * 将 JSON 数组转换为集合
     *
     * @param jsonArrayStr
     * @param clazz
     * @return
     * @throws Exception
     */
    public static <T> List<T> json2list(String jsonArrayStr, Class<T> clazz) throws Exception {
        JavaType javaType = getCollectionType(ArrayList.class, clazz);
        List<T> list = (List<T>) objectMapper.readValue(jsonArrayStr, javaType);
        return list;
    }

    /**
     * 获取泛型的 Collection Type
     *
     * @param collectionClass 泛型的Collection
     * @param elementClasses  元素类
     * @return JavaType Java类型
     * @since 1.0
     */
    public static JavaType getCollectionType(Class<?> collectionClass, Class<?>... elementClasses) {
        return objectMapper.getTypeFactory().constructParametricType(collectionClass, elementClasses);
    }

    /**
     * 将 Map 转换为 JavaBean
     *
     * @param map
     * @param clazz
     * @return
     */
    public static <T> T map2pojo(Map map, Class<T> clazz) {
        return objectMapper.convertValue(map, clazz);
    }

    /**
     * 将 Map 转换为 JSON
     *
     * @param map
     * @return
     */
    public static String mapToJson(Map map) {
        try {
            return objectMapper.writeValueAsString(map);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 将 JSON 对象转换为 JavaBean
     *
     * @param obj
     * @param clazz
     * @return
     */
    public static <T> T obj2pojo(Object obj, Class<T> clazz) {
        return objectMapper.convertValue(obj, clazz);
    }
}
```
utils.RegexpUtils

``` zephir
public class RegexpUtils {
    /**
     * 验证手机号
     */
    public static final String PHONE = "^((13[0-9])|(15[^4,\\D])|(18[0,5-9]))\\d{8}$";

    /**
     * 验证邮箱地址
     */
    public static final String EMAIL = "\\w+(\\.\\w)*@\\w+(\\.\\w{2,3}){1,3}";

    /**
     * 验证手机号
     * @param phone
     * @return
     */
    public static boolean checkPhone(String phone) {
        return phone.matches(PHONE);
    }

    /**
     * 验证邮箱
     * @param email
     * @return
     */
    public static boolean checkEmail(String email) {
        return email.matches(EMAIL);
    }
}
```
web.AbstractBaseController

``` aspectj
public abstract class AbstractBaseController<T extends AbstractBaseDomain> {

    // 用于动态获取配置文件的属性值
    private static final String ENVIRONMENT_LOGGING_LEVEL_MY_SHOP = "logging.level.com.ht.micro.record";

    @Resource
    protected HttpServletRequest request;
    @Resource
    protected HttpServletResponse response;

    @Autowired
    private ConfigurableApplicationContext applicationContext;

    @ModelAttribute
    public void initReqAndRes(HttpServletRequest request, HttpServletResponse response) {
        this.request = request;
        this.response = response;
    }

    /**
     * 请求成功
     * @param self
     * @param attribute
     * @return
     */
    protected AbstractBaseResult success(String self, T attribute) {
        return BaseResultFactory.getInstance(response).build(self, attribute);
    }

    /**
     * 请求成功
     * @param self
     * @param next
     * @param last
     * @param attributes
     * @return
     */
    protected AbstractBaseResult success(String self, int next, int last, List<T> attributes) {
        return BaseResultFactory.getInstance(response).build(self, next, last, attributes);
    }

    /**
     * 请求失败
     * @param title
     * @param detail
     * @return
     */
    protected AbstractBaseResult error(String title, String detail) {
        return error(HttpStatus.UNAUTHORIZED.value(), title, detail);
    }

    /**
     * 请求失败
     * @param code
     * @param title
     * @param detail
     * @return
     */
    protected AbstractBaseResult error(int code, String title, String detail) {
        return BaseResultFactory.getInstance(response).build(code, title, detail, applicationContext.getEnvironment().getProperty(ENVIRONMENT_LOGGING_LEVEL_MY_SHOP));
    }
}
```
## ht-micro-record-commons-domain
domain.TbUser
``` typescript
@Table(name = "tb_user")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class TbUser extends AbstractBaseDomain {

    /**
     * 用户名
     */
    @NotNull(message = "用户名不可为空")
    @Length(min = 5, max = 20, message = "用户名长度必须介于 5 和 20 之间")
    private String username;

    /**
     * 密码，加密存储
     */
    @JsonIgnore
    private String password;

    /**
     * 注册手机号
     */
    private String phone;

    /**
     * 注册邮箱
     */
    @NotNull(message = "邮箱不可为空")
    @Pattern(regexp = RegexpUtils.EMAIL, message = "邮箱格式不正确")
    private String email;

    /**
     * 获取用户名
     *
     * @return username - 用户名
     */
    public String getUsername() {
        return username;
    }

    /**
     * 设置用户名
     *
     * @param username 用户名
     */
    public void setUsername(String username) {
        this.username = username;
    }

    /**
     * 获取密码，加密存储
     *
     * @return password - 密码，加密存储
     */
    public String getPassword() {
        return password;
    }

    /**
     * 设置密码，加密存储
     *
     * @param password 密码，加密存储
     */
    public void setPassword(String password) {
        this.password = password;
    }

    /**
     * 获取注册手机号
     *
     * @return phone - 注册手机号
     */
    public String getPhone() {
        return phone;
    }

    /**
     * 设置注册手机号
     *
     * @param phone 注册手机号
     */
    public void setPhone(String phone) {
        this.phone = phone;
    }

    /**
     * 获取注册邮箱
     *
     * @return email - 注册邮箱
     */
    public String getEmail() {
        return email;
    }

    /**
     * 设置注册邮箱
     *
     * @param email 注册邮箱
     */
    public void setEmail(String email) {
        this.email = email;
    }
}
```
validator.BeanValidator

``` processing
@Component
public class BeanValidator {

    @Autowired
    private Validator validatorInstance;

    private static Validator validator;

    // 系统启动时将静态对象注入容器，不可用Autowire直接注入
    @PostConstruct
    public void init() {
        BeanValidator.validator = validatorInstance;
    }

    /**
     * 调用 JSR303 的 validate 方法, 验证失败时抛出 ConstraintViolationException.
     */
    private static void validateWithException(Validator validator, Object object, Class<?>... groups) throws ConstraintViolationException {
        Set constraintViolations = validator.validate(object, groups);
        if (!constraintViolations.isEmpty()) {
            throw new ConstraintViolationException(constraintViolations);
        }
    }

    /**
     * 辅助方法, 转换 ConstraintViolationException 中的 Set<ConstraintViolations> 中为 List<message>.
     */
    private static List<String> extractMessage(ConstraintViolationException e) {
        return extractMessage(e.getConstraintViolations());
    }

    /**
     * 辅助方法, 转换 Set<ConstraintViolation> 为 List<message>
     */
    private static List<String> extractMessage(Set<? extends ConstraintViolation> constraintViolations) {
        List<String> errorMessages = new ArrayList<>();
        for (ConstraintViolation violation : constraintViolations) {
            errorMessages.add(violation.getMessage());
        }
        return errorMessages;
    }

    /**
     * 辅助方法, 转换 ConstraintViolationException 中的 Set<ConstraintViolations> 为 Map<property, message>.
     */
    private static Map<String, String> extractPropertyAndMessage(ConstraintViolationException e) {
        return extractPropertyAndMessage(e.getConstraintViolations());
    }

    /**
     * 辅助方法, 转换 Set<ConstraintViolation> 为 Map<property, message>.
     */
    private static Map<String, String> extractPropertyAndMessage(Set<? extends ConstraintViolation> constraintViolations) {
        Map<String, String> errorMessages = new HashMap<>();
        for (ConstraintViolation violation : constraintViolations) {
            errorMessages.put(violation.getPropertyPath().toString(), violation.getMessage());
        }
        return errorMessages;
    }

    /**
     * 辅助方法, 转换 ConstraintViolationException 中的 Set<ConstraintViolations> 为 List<propertyPath message>.
     */
    private static List<String> extractPropertyAndMessageAsList(ConstraintViolationException e) {
        return extractPropertyAndMessageAsList(e.getConstraintViolations(), " ");
    }

    /**
     * 辅助方法, 转换 Set<ConstraintViolations> 为 List<propertyPath message>.
     */
    private static List<String> extractPropertyAndMessageAsList(Set<? extends ConstraintViolation> constraintViolations) {
        return extractPropertyAndMessageAsList(constraintViolations, " ");
    }

    /**
     * 辅助方法, 转换 ConstraintViolationException 中的 Set<ConstraintViolations> 为 List<propertyPath + separator + message>.
     */
    private static List<String> extractPropertyAndMessageAsList(ConstraintViolationException e, String separator) {
        return extractPropertyAndMessageAsList(e.getConstraintViolations(), separator);
    }

    /**
     * 辅助方法, 转换 Set<ConstraintViolation> 为 List<propertyPath + separator + message>.
     */
    private static List<String> extractPropertyAndMessageAsList(Set<? extends ConstraintViolation> constraintViolations, String separator) {
        List<String> errorMessages = new ArrayList<>();
        for (ConstraintViolation violation : constraintViolations) {
            errorMessages.add(violation.getPropertyPath() + separator + violation.getMessage());
        }
        return errorMessages;
    }

    /**
     * 服务端参数有效性验证
     *
     * @param object 验证的实体对象
     * @param groups 验证组
     * @return 验证成功：返回 null；验证失败：返回错误信息
     */
    public static String validator(Object object, Class<?>... groups) {
        try {
            validateWithException(validator, object, groups);
        } catch (ConstraintViolationException ex) {
            List<String> list = extractMessage(ex);
            list.add(0, "数据验证失败：");

            // 封装错误消息为字符串
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < list.size(); i++) {
                String exMsg = list.get(i);
                if (i != 0) {
                    sb.append(String.format("%s. %s", i, exMsg)).append(list.size() > 1 ? "<br/>" : "");
                } else {
                    sb.append(exMsg).append(list.size() > 1 ? "<br/>" : "");
                }
            }

            return sb.toString();
        }

        return null;
    }
}
```
## ht-micro-record-commons-mapper
├─src
│  └─main
│      ├─java
│      │  ├─com
│      │  │  └─ht
│      │  │      └─micro
│      │  │          └─record
│      │  │              └─commons
│      │  │                  └─mapper
│      │  │                      ├─baseMapper  
│      │  │                      └─dicMapper
│      │  └─tk
│      │      └─mybatis
│      │          └─mapper
│      └─resources
│          ├─baseMapper
│          └─dicMapper
mapper分开存储用来识别多数据源
tk.mybatis.mapper.MyMapper

``` php
public interface MyMapper<T> extends Mapper<T>, MySqlMapper<T> {
}
```
## ht-micro-record-commons-service
BaseCrudService

``` java
public interface BaseCrudService<T extends AbstractBaseDomain> {

    /**
     * 查询属性值是否唯一
     *
     * @param property
     * @param value
     * @return true/唯一，false/不唯一
     */
    default boolean unique(String property, String value) {
        return false;
    }

    /**
     * 保存
     *
     * @param domain
     * @return
     */
    default T save(T domain) {
        return null;
    }

    /**
     * 分页查询
     * @param domain
     * @param pageNum
     * @param pageSize
     * @return
     */
    default PageInfo<T> page(T domain, int pageNum, int pageSize) {
        return null;
    }
}
```
impl.BaseCrudServiceImpl

``` scala
public class BaseCrudServiceImpl<T extends AbstractBaseDomain, M extends MyMapper<T>> implements BaseCrudService<T> {

    @Autowired
    protected M mapper;

    private Class<T> entityClass = (Class<T>) ((ParameterizedType) getClass().getGenericSuperclass()).getActualTypeArguments()[0];

    @Override
    public boolean unique(String property, String value) {
        Example example = new Example(entityClass);
        example.createCriteria().andEqualTo(property, value);
        int result = mapper.selectCountByExample(example);
        if (result > 0) {
            return false;
        }
        return true;
    }

    @Override
    public T save(T domain) {
        int result = 0;
        Date currentDate = new Date();

        // 创建
        if (domain.getId() == null) { 

            /**
             * 用于自动回显 ID，领域模型中需要 @ID 注解的支持
             * {@link AbstractBaseDomain}
             */
            result = mapper.insertUseGeneratedKeys(domain);
        }

        // 更新
        else {
            result = mapper.updateByPrimaryKey(domain);
        }

        // 保存数据成功
        if (result > 0) {
            return domain;
        }

        // 保存数据失败
        return null;
    }

    @Override
    public PageInfo<T> page(T domain, int pageNum, int pageSize) {
        Example example = new Example(entityClass);
        example.createCriteria().andEqualTo(domain);

        PageHelper.startPage(pageNum, pageSize);
        PageInfo<T> pageInfo = new PageInfo<>(mapper.selectByExample(example));
        return pageInfo;
    }
}
```
TbUserService

``` java
public interface TbUserService extends BaseCrudService<TbUser> {

    TbUser getById(long id);
}
```
impl.TbUserServiceImpl

``` scala
@Service
public class TbUserServiceImpl extends BaseCrudServiceImpl<TbUser, TbUserMapper> implements TbUserService {

    @Autowired
    private TbUserMapper tbUserMapper;

    public TbUser getById(long id){
        return tbUserMapper.selectByPrimaryKey(id);
    }

}
```
## ht-micro-record-service-user
``` dust
 <parent>
        <groupId>com.htdc</groupId>
        <artifactId>ht-micro-record-dependencies</artifactId>
        <version>1.0.0-SNAPSHOT</version>
        <relativePath/>
    </parent>

    <artifactId>ht-micro-record-service-user</artifactId>
    <packaging>jar</packaging>

    <name>ht-micro-record-service-user</name>
    <url>http://www.htdatacloud.com/</url>
    <inceptionYear>2019-Now</inceptionYear>

    <dependencies>
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
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
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
                    <mainClass>com.ht.micro.record.UserServiceApplication</mainClass>
                </configuration>
            </plugin>

        </plugins>
    </build>
```
application.yml
配置多数据源及RocketMQ
``` less
spring:
  cloud:
    stream:
      rocketmq:
        binder:
          name-server: 192.168.2.7:9876
      bindings:
        output:
          content-type: application/json
          destination: topic-email
          producer:
            group: group-email
  datasource:
      base:
        #监控统计拦截的filters
        filters: stat
        type: com.alibaba.druid.pool.DruidDataSource
        jdbc-url: jdbc:mysql://192.168.2.7:185/ht_micro_record?useUnicode=true&characterEncoding=UTF-8&useSSL=false
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
bootstrap.properties

``` stylus
spring.application.name=ht-micro-record-service-user-config
spring.cloud.nacos.config.file-extension=yaml
spring.cloud.nacos.config.server-addr=192.168.2.7:8848,192.168.2.7:8849,192.168.2.7:8850
logging.level.com.ht.micro.record=DEBUG
```
nacos配置

``` less
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
        dashboard: 192.168.2.7:190


server:
  port: 9506

management:
  endpoints:
    web:
      exposure:
        include: "*"
```
com.ht.micro.record.UserServiceApplication

``` less
@SpringBootApplication(scanBasePackages = "com.ht.micro.record",exclude = {DataSourceAutoConfiguration.class, DataSourceTransactionManagerAutoConfiguration.class, MybatisAutoConfiguration.class})
@EnableDiscoveryClient
@EnableBinding({Source.class})
@MapperScan(basePackages = "com.ht.micro.record.commons.mapper")
@EnableAsync
@EnableSwagger2
public class UserServiceApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```
com.ht.micro.record.user.config.BaseMybatisConfig

``` kotlin
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
com.ht.micro.record.user.config.DicMybatisConfig

``` kotlin
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
com.ht.micro.record.user.service.UserService

``` java
@Service
public class UserService {


    @Autowired
    private MessageChannel output;


    // @EnableAsync在Application中开启异步
    @Async
    public void sendEmail(TbUser tbUser) throws Exception {
        output.send(MessageBuilder.withPayload(MapperUtils.obj2json(tbUser)).build());
    }


}
```
com.ht.micro.record.user.controller.UserController

``` kotlin
@RestController
@RequestMapping(value = "user")
public class UserController extends AbstractBaseController<TbUser> {
    @Autowired
    private TbUserService tbUserService;
    @Autowired
    private UserService userService;

    // http://localhost:9506/user/2
    //
    @ApiOperation(value = "查询用户", notes = "根据id获取用户名")
    @GetMapping(value = {"{id}"})
    public String getName(@PathVariable long id){
        return tbUserService.getById(id).getUsername();
    }

    @ApiOperation(value = "用户注册", notes = "参数为实体类，注意用户名和邮箱不要重复")
    @PostMapping(value = "reg")
    public AbstractBaseResult reg(@ApiParam(name = "tbUser", value = "用户模型") TbUser tbUser) {
        // 数据校验
        String message = BeanValidator.validator(tbUser);
        if (StringUtils.isNotBlank(message)) {
            return error(message, null);
        }


        // 验证密码是否为空
        if (StringUtils.isBlank(tbUser.getPassword())) {
            return error("密码不可为空", null);
        }


        // 验证用户名是否重复
        if (!tbUserService.unique("username", tbUser.getUsername())) {
            return error("用户名已存在", null);
        }


        // 验证邮箱是否重复
        if (!tbUserService.unique("email", tbUser.getEmail())) {
            return error("邮箱重复，请重试", null);
        }


        // 注册用户
        try {
            tbUser.setPassword(DigestUtils.md5DigestAsHex(tbUser.getPassword().getBytes()));
            TbUser user = tbUserService.save(tbUser);
            if (user != null) {
                userService.sendEmail(user);
                response.setStatus(HttpStatus.CREATED.value());
                return success(request.getRequestURI(), user);
            }
        } catch (Exception e) {
            // 这里补一句，将 RegService 中的异常抛到 Controller 中，这样可以打印出调试信息
            return error(HttpStatus.INTERNAL_SERVER_ERROR.value(), "注册邮件发送失败", e.getMessage());
        }


        // 注册失败
        return error("注册失败，请重试", null);
    }
}
```

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
http://192.168.2.7:193/#/monitor/dashboard 查看skywalking的链路追踪 

详情见：
https://github.com/OneJane/blog



