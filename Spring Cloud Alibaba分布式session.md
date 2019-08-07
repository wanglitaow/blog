@[TOC]
com.ht.micro.record.commons.domain.User

``` groovy
@Data
public class User implements Serializable {

    private long userId;
    private String username;
    private String password;

}
```
# ht-micro-record-service-sms
pom.xml

``` 
        <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
```
SmsServiceApplication.java

``` 
@EnableRedisHttpSession(maxInactiveIntervalInSeconds= 1800) //开启redis session支持,并配置session过期时间
```
UserController.java

``` 
@Controller
@RequestMapping(value = "user")
public class UserController extends AbstractBaseController<TbUser> {

    @RequestMapping
    public String index() {
        return "index";
    }

    @RequestMapping("home")
    public String home() {
        return "home";
    }


    @PostMapping("login")
    public String login(User user, HttpSession session) {
        // 随机生成用户id
        user.setUserId(Math.round(Math.floor(Math.random() * 10 * 1000)));
        // 将用户信息保存到id中
        session.setAttribute("USER", user);
        return "home";
    }

    @PostMapping("logout")
    public String logout(HttpSession session) {
        session.removeAttribute("USER");
        session.invalidate();
        return "home";
    }

}
```
templates/home.ftl

``` 
<!doctype html>
<html lang="en">
<head>
    <title>主页面</title>
</head>
<body>
    <h5>登录用户: ${Session["USER"].username} </h5>
    <h5>用户编号: ${Session["USER"].userId} </h5>
</body>
</html>
```
templates/index.ftl

``` 
<!doctype html>
<html lang="en">
<head>
    <title>登录页面</title>
</head>
<body>
    <form action="/user/login" method="post">
        用户：<input type="text" name="username"><br/>
        密码：<input type="password" name="password"><br/>
        <button type="submit">登录</button>
    </form>
</body>
</html>
```
# ht-micro-record-service-user
 pom.xml

``` xml
       <dependency>
            <groupId>org.springframework.session</groupId>
            <artifactId>spring-session-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-freemarker</artifactId>
        </dependency>
```
application.yml
所有模块加入redis配置
``` yaml
redis:
    host: 127.0.0.1
    port: 6379
    jedis:
      pool:
        # 连接池最大连接数,使用负值表示无限制。
        max-active: 8
        # 连接池最大阻塞等待时间,使用负值表示无限制。
        max-wait: -1s
        # 连接池最大空闲数,使用负值表示无限制。
        max-idle: 8
        # 连接池最小空闲连接，只有设置为正值时候才有效
        min-idle: 1
    timeout: 300ms
    session:
      # session 存储方式 支持redis、mongo、jdbc、hazelcast
      store-type: redis

  # 如果是集群节点 采用如下配置指定节点
  #spring.redis.cluster.nodes
```
templates/index.ftl

``` xml
<!doctype html>
<html lang="en">
<head>
    <title>登录页面</title>
</head>
<body>
    <form action="/user/login" method="post">
        用户：<input type="text" name="username"><br/>
        密码：<input type="password" name="password"><br/>
        <button type="submit">登录</button>
    </form>
</body>
</html>
```
templates/home.ftl

``` 
<!doctype html>
<html lang="en">
<head>
    <title>主页面</title>
</head>
<body>
    <h5>登录用户: ${Session["USER"].username} </h5>
    <h5>用户编号: ${Session["USER"].userId} </h5>
</body>
</html>
```
UserServiceApplication.java

``` kotlin
@EnableRedisHttpSession(maxInactiveIntervalInSeconds= 1800) //开启redis session支持,并配置session过期时间
```
UserController.java

``` kotlin
@Controller
@RequestMapping(value = "user")
public class UserController extends AbstractBaseController<TbUser> {
    @Autowired
    private TbUserService tbUserService;
    @Autowired
    private UserService userService;

    @RequestMapping
    public String index() {
        return "index";
    }

    @RequestMapping("home")
    public String home() {
        return "home";
    }


    @PostMapping("login")
    public String login(User user, HttpSession session) {
        // 随机生成用户id
        user.setUserId(Math.round(Math.floor(Math.random() * 10 * 1000)));
        // 将用户信息保存到id中
        session.setAttribute("USER", user);
        return "home";
    }

    @PostMapping("logout")
    public String logout(HttpSession session) {
        session.removeAttribute("USER");
        session.invalidate();
        return "home";
    }
}	
```
http://localhost:9507/user 登陆后，进入http://localhost:9507/user/login 页面查看用户信息
手动进入http://localhost:9506/user/home 查看相同用户信息，User实体位置在两个服务中保持一致。
详情见：
https://github.com/OneJane/blog