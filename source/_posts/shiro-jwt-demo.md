---
title: 基于Shiro | JWT整合WxJava实现微信小程序登录
comments: true
auto_excerpt: true
toc: true
date: 2021-03-26 21:54:59
tags: 
    - Java
    - 微信小程序
categories:
    - 原创
    - 后端

description: 最近在做毕业设计，涉及到微信小程序的开发，要求前端小程序用户使用微信身份登录，登陆成功后，后台返回自定义登录状态token给小程序，后续小程序发送API请求都需要携带token才能访问后台数据。
---

掘金原文体验更加[👉👉戳这里](https://juejin.cn/post/6942706988534988836)

## 前言

最近在做毕业设计，涉及到微信小程序的开发，要求前端小程序用户使用微信身份登录，登陆成功后，后台返回自定义登录状态token给小程序，后续小程序发送API请求都需要携带token才能访问后台数据。


本文是对接微信小程序，实现自定义登录状态的一个完整示例，实现了小程序的自定义登陆，将自定义登陆态token返回给小程序作为登陆凭证。用户的信息保存在数据库中，登陆态token缓存在redis中。涉及的技术栈：
* SpringBoot -> 后端基础环境
* Shiro -> 安全框架
* JWT -> 加密token
* MySQL -> 主库，存储业务数据
* MyBatis-Plus -> 操作数据库
* Redis -> 缓存token和其他热点数据
* Lombok -> 简化开发
* FastJson -> json消息处理
* RestTemplate -> 优雅的处理web请求

项目GitHub地址：https://github.com/gongsir0630/shiro-jwt-demo

## 特性
* 基于WxJava对接微信小程序，实现用户登录、消息处理
* 支持Shiro注解编程，保持高度的灵活性
* 使用JWT进行校验，完全实现无状态鉴权
* 使用Redis存储自定义登陆态token，支持过期时间
* 支持跨域请求

## 准备工作
**基础知识预备：**
* 具备SpringBoot基础知识并且会使用基本注解；
* 了解JWT（Json Web Token）的基本概念，并且会简单操作JWT的 [JAVA SDK](https://github.com/auth0/java-jwt)；
* 了解Shiro的基本概念：Subject、Realm、SecurityManager等（建议去官网学习一下）

**其他说明：**

本文只对shiro和jwt整合进行介绍说明，具体的微信登录实现是使用`RestTemplate`调用我自己的`wx-java-miniapp`项目，该项目基于`WxJava`实现，支持多个小程序登录、消息处理。

**本文使用以下调用处理即可：**
```java
// 1. todo: 微信登录: code + appid -> openId + session_key
// appid: 从配置文件读取
MultiValueMap<String, Object> request = new LinkedMultiValueMap<>();
// 参数封装, 微信登录需要以下参数
request.add("code", code);
// eg: http://localhost:8081/wx/user/{appid}/login
String path = url+"/user/"+appid+"/login";
// 请求
JSONObject dto = restTemplate.postForObject(path, request, JSONObject.class);
log.info("--->>>来自[{}]的返回 = [{}]",path,dto);

// 2. todo: 使用openId和session_key生成自定义登录状态 -> token
```
**项目地址：**

* [wx-java-miniapp -> 可以直接部署使用](https://github.com/gongsir0630/wx-java-miniapp)
* [WxJava -> 官方SDK](https://github.com/Wechat-Group/WxJava)


## 整体思路
先了解一下小程序官方登录流程，[官方说明戳这里](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html)

![小程序登录流程](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c7b2c5fa55c46fbbd41976635b20807~tplv-k3u1fbpfcp-watermark.image)

1. 小程序调用`wx.login()`得到`code`，将code发送到后台，后台通过`wx-java-miniapp`获取到用户的`openId`和`session_key`；
2. 后台通过jwt工具生成自定义用户状态信息`token`，并且后台在数据库中查询`openId`判断是否存在，根据查询结果封装不同的消息，最后连同`token`一起返回给小程序；
3. 之后用户访问每一个需要权限的API请求必须在`header`中添加`Authorization`字段，后台会进行`token`的校验，如果有误会直接返回`401`。

## token加密说明
* 使用`uuid`随机生成一个jwt-id
* 将用户的`openId`、`session_key`连同`jwt-id`一起，使用小程序的`appid`进行签名加密并设置过期时间，最终生成`token`
* 将`"JWT-SESSION-"+jwt-id`和`token`以key-value的形式存入`redis`中，并设置相同的过期时间

## token校验说明
- 解析token中jwt-id
- 以`"JWT-SESSION-"+jwt-id`为key从redis中获取redisToken
- 解析`redisToken`的携带信息，重新以相同的方式生成验证器，同`token`进行校验比对

## 项目实现
- 项目数据库使用`MySQL`作为作为主库，如果是`clone`的项目，请在运行之前准备好相应的数据库，并修改配置信息。
- 项目使用了redis缓存，运行前请在本地安装`redis`，使用默认配置即可，无需修改。
- 项目中使用了`lombok`简化开发，请在idea或者eclipse安装lombok插件。

### 创建Maven项目
新建一个SpringBoot项目，修改pom文件，添加相关dependency:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.github.gongsir0630</groupId>
    <artifactId>shiro-jwt-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>shiro-jwt-demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <spring-boot.version>2.3.7.RELEASE</spring-boot.version>
    </properties>

    <dependencies>
        <!-- shiro: 用户认证\接口鉴权 -->
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.4.0</version>
        </dependency>
        <!-- jwt: token认证 -->
        <dependency>
            <groupId>com.auth0</groupId>
            <artifactId>java-jwt</artifactId>
            <version>3.4.1</version>
        </dependency>
        <!-- redis: 数据缓存 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!-- 引入fastjson -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.47</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-json</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- druid数据库连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <!-- mybatis-plus: 操作数据库 -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.4.2</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <!-- mysql驱动-->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- 工具: 简化model开发 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- 单元测试工具 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.3.7.RELEASE</version>
                <configuration>
                    <mainClass>com.github.gongsir0630.shirodemo.ShiroJwtDemoApplication</mainClass>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```
>注意JDK版本：1.8

### 相关配置 | 工具准备
配置你的application.yml ，主要是配置你的小程序appid和url，还有你的数据库和redis。
```yml
# 设置日志级别
logging:
    level:
        org.springframework.web: info
        com.github.gongsir0630.shirodemo: debug
# dev环境配置文件
spring:
    # 数据库相关配置信息: 无需再本地安装mysql,使用yzhelp.top云端数据库
    datasource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://localhost:3306/shiro-jwt-demo
        username: username
        password: password
    # redis 配置信息: 在本地安装redis
    redis:
        host: 127.0.0.1
        port: 6379
        database: 0

---
# 服务启动的端口号
server:
  port: 8080

---
# 微信小程序配置 appid / url
wx:
    # 小程序AppId
    appid: appid
    # 自研小程序接口调用地址
    url: http://localhost:8081/wx
```
>说明：<br/>appid: 当前小程序的appid<br/>url: wx-java-miniapp项目接口地址

#### 配置fastJson
在启动类中配置`fastJson` -> **ShiroJwtDemoApplication.java**

```java
/**
 * @author gongsir <a href="https://github.com/gongsir0630">程序员Kyle✨</a>
 * 描述: Spring Boot 工程启动类,可以直接点击下面的main方法运行程序
 */

@SpringBootApplication
public class ShiroJwtDemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(ShiroJwtDemoApplication.class, args);
    }

    /**
     * fastjson 配置注入: 使用阿里巴巴的 fastjson 处理 json 信息
     * @return HttpMessageConverters
     */
    @Bean
    public HttpMessageConverters fastJsonHttpMessageConverters() {
        // 消息转换对象
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        // fastjson 配置
        FastJsonConfig config = new FastJsonConfig();
        config.setSerializerFeatures(SerializerFeature.PrettyFormat);
        config.setDateFormat("yyyy-MM-dd");
        // 配置注入消息转换器
        converter.setFastJsonConfig(config);
        // 让 spring 使用自定义的消息转换器
        return new HttpMessageConverters(converter);
    }
}

```
#### 配置Redis
配置Redis -> **RedisConfig.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 14:15
 * 你的指尖,拥有改变世界的力量
 * 描述: Redis配置
 * EnableCaching: 开启缓存
 */
@Configuration
@EnableCaching
public class RedisConfig {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.create(factory);
    }
}
```
#### 配置RestTemplate
配置RestTemplate -> **RestTemplateConfig.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 14:10
 * 你的指尖,拥有改变世界的力量
 * 描述: RestTemplate的配置类
 */
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(ClientHttpRequestFactory factory) {
        return new RestTemplate(factory);
    }

    @Bean
    public ClientHttpRequestFactory simpleClientHttpRequestFactory() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        // 连接超时时间设置为10秒
        factory.setConnectTimeout(1000 * 10);
        // 读取超时时间为单位为60秒
        factory.setReadTimeout(1000 * 60);
        return factory;
    }
}
```
#### 返回集封装

**CodeMsg.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 20:17
 * 你的指尖,拥有改变世界的力量
 * 描述: code和msg封装
 */
public class CodeMsg {
    private final int code;
    private final String msg;

    public static CodeMsg SUCCESS=new CodeMsg(0,"success");

    public static CodeMsg LOGIN_FAIL = new CodeMsg(-1,"code2session failure, please try aging");

    public static CodeMsg NO_USER = new CodeMsg(1000,"user not found");
    public static CodeMsg SESSION_KEY_ERROR = new CodeMsg(1001,"sessionKey is invalid");
    public static CodeMsg TOKEN_ERROR = new CodeMsg(1002,"token is invalid");
    public static CodeMsg SHIRO_ERROR = new CodeMsg(1003,"token is invalid");

    public CodeMsg(int code, String msg) {
        this.code=code;
        this.msg=msg;
    }

    public int getCode() {
        return code;
    }

    public String getMsg() {
        return msg;
    }

    @Override
    public String toString() {
        return "CodeMsg{" +
                "code=" + code +
                ", msg='" + msg + '\'' +
                '}';
    }

}
```
**Result.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 18:45
 * 你的指尖,拥有改变世界的力量
 * 描述:
 * 输出结果的封装
 * 只要get不要set,进行更好的封装
 * @param <T> data泛型
 */
public class Result<T> {

    private int code;
    private String msg;
    private T data;


    private Result(T data){
        this.code=0;
        this.msg="success";
        this.data=data;
    }

    private Result(CodeMsg mg, T data) {
        if (mg==null){
            return;
        }
        this.code=mg.getCode();
        this.msg=mg.getMsg();
        this.data=data;
    }


    /**
     * 成功时
     * @param <T> data泛型
     * @return Result
     */
    public static <T>  Result<T>  success(T data){
        return new Result<T>(data);
    }

    /**
     * 失败
     * @param <T> data泛型
     * @return Result
     */
    public static <T>  Result<T>  fail(CodeMsg mg, T data){
        return new Result<T>(mg,data);
    }

    public int getCode() {
        return code;
    }


    public String getMsg() {
        return msg;
    }


    public T getData() {
        return data;
    }
}
```
#### 异常封装与处理
自定义异常 -> **ApiAuthException.java**

```java
import com.github.gongsir0630.shirodemo.controller.res.CodeMsg;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 20:24
 * 你的指尖,拥有改变世界的力量
 * 描述: 自定义异常, 用于处理Api认证失败异常信息保存
 */
public class ApiAuthException extends RuntimeException {
    private CodeMsg codeMsg;

    public ApiAuthException() {
        super();
    }

    public ApiAuthException(CodeMsg codeMsg) {
        super(codeMsg.getMsg());
        this.codeMsg = codeMsg;
    }

    public CodeMsg getCodeMsg() {
        return codeMsg;
    }

    public void setCodeMsg(CodeMsg codeMsg) {
        this.codeMsg = codeMsg;
    }
}
```
全局异常处理 -> **AppExceptionHandler.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 15:49
 * 你的指尖,拥有改变世界的力量
 * 描述: 全局异常处理
 */
@RestControllerAdvice
@Slf4j
public class AppExceptionHandler {

    /**
     * 处理 Shiro 异常
     * @param e 异常信息
     * @return json
     */
    @ExceptionHandler({ShiroException.class})
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public ResponseEntity<Result<JSONObject>> handShiroException(ShiroException e) {
        log.error("--->>> 捕捉到 [ApiAuthException] 异常: {}", e.getMessage());
        return new ResponseEntity<>(Result.fail(CodeMsg.SHIRO_ERROR,null), HttpStatus.UNAUTHORIZED);
    }

    /**
     * 处理 自定义ApiAuthException异常
     * @param e 异常信息
     * @return json
     */
    @ExceptionHandler({ApiAuthException.class})
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public ResponseEntity<Result<JSONObject>> handApiAuthException(ApiAuthException e) {
        log.error("--->>> 捕捉到 [ApiAuthException] 异常: {},{}",e.getCodeMsg().getCode(),e.getCodeMsg().getMsg() );
        return new ResponseEntity<>(Result.fail(e.getCodeMsg(),null), HttpStatus.UNAUTHORIZED);
    }
}
```

### 准备数据源
- 数据库：shiro-jwt-demo
- 数据表：user
![示例数据库表结构](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b52785aad974131a859d2a51feaa0ca~tplv-k3u1fbpfcp-watermark.image)

> 注意：这里是业务数据库，也就是我们小程序用户信息都由我们自己存储，第一次默认使用微信公开信息注册，之后用户可以自行更新这些信息，和微信信息独立开。

创建对应的实体类 -> **User.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/21 23:44
 * 你的指尖,拥有改变世界的力量
 * 描述: 业务用户信息
 */
@Data
@TableName("user")
public class User {
    /**
     * 主键,数据库字段为user_id -> userId == openId
     */
    @TableId(value = "user_id",type = IdType.INPUT)
    private String userId;
    private String name;
    private String photo;
    private String sex;
    private String grade;
    private String college;
    private String contact;
}
```
使用MyBatis-plus创建mapper接口 -> **UserMapper.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 19:44
 * 你的指尖,拥有改变世界的力量
 * 描述: User类mapper接口,继承自BaseMapper(已经实现User的CRUD)
 */
@Mapper
public interface UserMapper extends BaseMapper<User> {
}
```
MyBatis-Plus配置 -> **MybatisPlusConfig.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/19 17:15
 * 你的指尖,拥有改变世界的力量
 * 描述: MyBatis-Plus插件配置
 */

@Configuration
@MapperScan("com.github.gongsir0630.shirodemo.mapper")
public class MybatisPlusConfig {
}
```
创建User业务接口，这里仅仅演示login -> **UserService.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 19:49
 * 你的指尖,拥有改变世界的力量
 * 描述: 用户接口
 */
public interface UserService extends IService<User> {
    /**
     * 登录
     * @param jsCode 小程序code
     * @return 登录信息: 包含token
     */
    Map<String, String> login(String jsCode);
}
```
再创建一个微信登录信息对象，主要用作接收微信的openid和session_key，以及用作shiro认证 -> **WxAccount.java**

```java
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 19:58
 * 你的指尖,拥有改变世界的力量
 * 描述: 微信认证信息
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class WxAccount {
    private String openId;
    private String sessionKey;
}
```
> 注意：该类不会用于业务信息交互，所以不需要Mapper与db交互。

微信登录接口，在这里实现与微信服务器的信息交互 -> **WxAccountService.java**

```java
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/22 20:06
 * 你的指尖,拥有改变世界的力量
 * 描述: 微信接口
 */
public interface WxAccountService {
    /**
     * 微信小程序用户登陆，完整流程可参考下面官方地址，本例中是按此流程开发
     * https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html
     * 1 . 微信小程序端传入code。
     * 2 . 通过wx-java-miniapp项目调用微信code2session接口获取openid和session_key
     *
     * @param code 小程序端 调用 wx.login 获取到的code,用于调用 微信code2session接口
     * @return JSONObject: 包含openId和sessionKey
     */
    WxAccount login(String code);
}
```
接口实现逻辑：
1. 从配置文件读取`appid`和`url`；
2. 凭借目标请求地址`path`，例如登录是 `{url}/wx/user/{appid}/login`；
3. 参数封装，封装来自小程序的`code`；
4. 使用`RestTemplate`发起登录请求；
5. 处理返回集。

代码实现 -> **WxAccountServiceImpl.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 16:12
 * 你的指尖,拥有改变世界的力量
 * 描述: 微信接口实现: 用 restTemplate 调用 [wxApp] 应用的接口
 */
@Service
@Slf4j
public class WxAccountServiceImpl implements WxAccountService {

    @Value("${wx.appid}")
    private String appid;
    @Value("${wx.url}")
    private String url;

    @Resource
    private RestTemplate restTemplate;

    @Override
    public WxAccount login(String code) {
        // todo: 微信登录: code + appid -> openId + session_key
        // appid: 从配置文件读取
        MultiValueMap<String, Object> request = new LinkedMultiValueMap<>();
        // 参数封装, 微信登录需要以下参数
        request.add("code", code);
        // eg: http://localhost:8081/wx/user/{appid}/login
        String path = url+"/user/"+appid+"/login";
        // 请求
        JSONObject dto = restTemplate.postForObject(path, request, JSONObject.class);
        log.info("--->>>来自[{}]的返回 = [{}]",path,dto);
        int errCode = -1;
        if (dto != null ) {
            errCode = Integer.parseInt(dto.get("code").toString());
        } else {
            throw new ApiAuthException(CodeMsg.LOGIN_FAIL);
        }
        if (0 != errCode) {
            throw new ApiAuthException(new CodeMsg(Integer.parseInt(dto.get("code").toString()),
                dto.get("msg").toString()));
        }
        // code2session success
        JSONObject data = dto.getJSONObject("data");
        return JSON.toJavaObject(data, WxAccount.class);
    }
}
```
### 构建JWT

jwt工具类，用于生成token签名, token校验 -> **JwtUtil.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 10:26
 * 你的指尖,拥有改变世界的力量
 * 描述: jwt工具类: 生成token签名, token校验
 */
@Component
@SuppressWarnings("All")
public class JwtUtil {
    /**
     * 过期时间: 2小时
     */
    private static final long EXPIRE_TIME = 7200;
    /**
     * 使用 appid 签名
     */
    @Value("${wx.appid}")
    private String appsecret;

    @Autowired
    private StringRedisTemplate redisTemplate;

    /**
     * 根据微信用户登陆信息创建 token
     * 使用`uuid`随机生成一个jwt-id
     * 将用户的`openId`、`session_key`连同`jwt-id`一起，使用小程序的`appid`进行签名加密并设置过期时间，最终生成`token`
     * 将`"JWT-SESSION-"+jwt-id`和`token`以key-value的形式存入`redis`中，并设置相同的过期时间
     * 注 : 这里的token会被缓存到redis中,用作为二次验证
     * redis里面缓存的时间应该和jwt token的过期时间设置相同
     *
     * @param wxAccount 微信用户信息
     * @return 返回 jwt token
     */
    public String sign(WxAccount account) {
        //JWT 随机ID,做为redis验证的key
        String jwtId = UUID.randomUUID().toString();
        //1 . 加密算法进行签名得到token
        Algorithm algorithm = Algorithm.HMAC256(appsecret);
        String token = JWT.create()
                .withClaim("openId", account.getOpenId())
                .withClaim("sessionKey", account.getSessionKey())
                .withClaim("jwt-id",jwtId)
                .withExpiresAt(new Date(System.currentTimeMillis() + EXPIRE_TIME * 1000))
                .sign(algorithm);
        //2 . Redis缓存JWT, 注 : 请和JWT过期时间一致
        redisTemplate.opsForValue().set("JWT-SESSION-"+jwtId, token, EXPIRE_TIME, TimeUnit.SECONDS);
        return token;
    }

    /**
     * token 检验
     * @param token
     * @return bool
     */
    public boolean verify(String token) {
        try {
            //1 . 根据token解密，解密出jwt-id , 先从redis中查找出redisToken，匹配是否相同
            String redisToken = redisTemplate.opsForValue().get("JWT-SESSION-" + getClaimsByToken(token).get("jwt-id").asString());
            if (!token.equals(redisToken)) {
                return Boolean.FALSE;
            }
            //2 . 得到算法相同的JWTVerifier
            Algorithm algorithm = Algorithm.HMAC256(appsecret);
            JWTVerifier verifier = JWT.require(algorithm)
                    .withClaim("openId", getClaimsByToken(redisToken).get("openId").asString())
                    .withClaim("sessionKey", getClaimsByToken(redisToken).get("sessionKey").asString())
                    .withClaim("jwt-id",getClaimsByToken(redisToken).get("jwt-id").asString())
                    .build();
            //3 . 验证token
            verifier.verify(token);
            //4 . Redis缓存JWT续期
            redisTemplate.opsForValue().set("JWT-SESSION-" + getClaimsByToken(token).get("jwt-id").asString(),
                    redisToken,
                    EXPIRE_TIME,
                    TimeUnit.SECONDS);
            return Boolean.TRUE;
        } catch (Exception e) {
            //捕捉到任何异常都视为校验失败
            return Boolean.FALSE;
        }
    }

    /**
     * 从token解密信息
     * @param token token
     * @return
     * @throws JWTDecodeException
     */
    public Map<String, Claim> getClaimsByToken(String token) throws JWTDecodeException {
        return JWT.decode(token).getClaims();
    }
}
```
### Realm配置

创建JwtToken，用于shiro鉴权，需要实现`AuthenticationToken` -> **JwtToken.java**

```java
/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 10:48
 * 你的指尖,拥有改变世界的力量
 * 描述: 鉴权用的token，需要实现 AuthenticationToken
 */
@Data
@AllArgsConstructor
public class JwtToken implements AuthenticationToken {
    private String token;
    @Override
    public Object getPrincipal() {
        return token;
    }

    @Override
    public Object getCredentials() {
        return token;
    }
}
```

自定义Shiro的Realm配置，需要在Realm中实现我们自定义的登陆及授权逻辑 -> **ShiroRealm.java**

```java
import com.github.gongsir0630.shirodemo.controller.res.CodeMsg;
import com.github.gongsir0630.shirodemo.exception.ApiAuthException;
import com.github.gongsir0630.shirodemo.wx.util.JwtUtil;
import com.github.gongsir0630.shirodemo.wx.vo.JwtToken;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.SimpleAuthenticationInfo;
import org.apache.shiro.authc.credential.CredentialsMatcher;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.realm.Realm;
import org.apache.shiro.subject.PrincipalCollection;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 15:28
 * 你的指尖,拥有改变世界的力量
 * 描述: Realm 的一个配置管理类 allRealm()方法得到所有的realm
 */
@Component
@Slf4j
public class ShiroRealm {
    @Resource
    private JwtUtil jwtUtil;
    
    /**
     * 封装所有自定义的realm规则链 -> shiro配置中会将规则注入到shiro的securityManager
     * @return 所有自定义的realm规则
     */
    public List<Realm> allRealm() {
        List<Realm> realmList = new LinkedList<>();
        realmList.add(authorizingRealm());
        return Collections.unmodifiableList(realmList);
    }

    /**
     * 自定义 JWT的 Realm
     * 重写 Realm 的 supports() 方法是通过 JWT 进行登录判断的关键
     */
    private AuthorizingRealm authorizingRealm() {
        AuthorizingRealm realm = new AuthorizingRealm() {
            /**
             * 当需要检测 用户权限 时调用此方法，例如checkRole,checkPermission之类的
             * 根据业务需求自行编写验证逻辑
             * @param principalCollection == token
             */
            @Override
            protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
                String token = principalCollection.toString();
                log.info("--->>>PrincipalCollection: [{}]",token);
                // todo: 自定义权限验证, 比如role和permission验证
                return new SimpleAuthorizationInfo();
            }

            /**
             * 默认使用此方法进行用户名正确与否校验: 验证token逻辑
             */
            @Override
            protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
                String jwtToken = (String) authenticationToken.getCredentials();
                String openId = jwtUtil.getClaimsByToken(jwtToken).get("openId").asString();
                String sessionKey = jwtUtil.getClaimsByToken(jwtToken).get("sessionKey").asString();
                if (null == openId || "".equals(openId)) {
                    throw new ApiAuthException(CodeMsg.NO_USER);
                }
                if (null == sessionKey || "".equals(sessionKey)) {
                    throw new ApiAuthException(CodeMsg.SESSION_KEY_ERROR);
                }
                if (!jwtUtil.verify(jwtToken)) {
                    throw new ApiAuthException(CodeMsg.TOKEN_ERROR);
                }
                // 将 openId 和 sessionKey 装配到subject中
                // 在 Controller 中使用 SecurityUtils.getSubject().getPrincipal() 即可获取用户openId
                return new SimpleAuthenticationInfo(openId,sessionKey,this.getClass().getName());
            }

            /**
             * 注意坑点 : 必须重写此方法，不然Shiro会报错
             * 因为创建了 JWTToken 用于替换Shiro原生 token,所以必须在此方法中显式的进行替换，否则在进行判断时会一直失败
             */
            @Override
            public boolean supports(AuthenticationToken token) {
                return token instanceof JwtToken;
            }
        };
        realm.setCredentialsMatcher(credentialsMatcher());
        return realm;
    }

    /**
     * 注意 : 密码校验, 这里因为是JWT形式,就无需密码校验和加密,直接让其返回为true(如果不设置的话,该值默认为false,即始终验证不通过)
     */
    private CredentialsMatcher credentialsMatcher() {
        // 实现boolean doCredentialsMatch(AuthenticationToken var1, AuthenticationInfo var2);
        return (authenticationToken, authenticationInfo) -> true;
    }
}
```
### 重写filter
所有的请求都会先经过`Filter`，所以我们继承官方的`BasicHttpAuthenticationFilter`，并且重写方法即可 -> **JwtFilter.java**

```java
import com.github.gongsir0630.shirodemo.wx.vo.JwtToken;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.RequestMethod;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 10:58
 * 你的指尖,拥有改变世界的力量
 * 描述: JWT核心过滤器配置
 * 所有的请求都会先经过Filter，继承官方的BasicHttpAuthenticationFilter，并且重写鉴权的方法
 * 执行流程 preHandle->isAccessAllowed->isLoginAttempt->executeLogin
 */
@Slf4j
public class JwtFilter extends BasicHttpAuthenticationFilter {
    /**
     * 跨域支持
     * @param request 请求
     * @param response 相应
     * @return bool
     * @throws Exception 异常
     */
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        httpServletResponse.setHeader("Access-control-Allow-Origin", httpServletRequest.getHeader("Origin"));
        httpServletResponse.setHeader("Access-Control-Allow-Methods", "GET,POST,OPTIONS,PUT,DELETE");
        httpServletResponse.setHeader("Access-Control-Allow-Headers", httpServletRequest.getHeader("Access-Control-Request-Headers"));
        // 跨域时会首先发送一个option请求，这里我们给option请求直接返回正常状态
        if (httpServletRequest.getMethod().equals(RequestMethod.OPTIONS.name())) {
            httpServletResponse.setStatus(HttpStatus.OK.value());
            return false;
        }
        return super.preHandle(request, response);
    }

    @Override
    protected boolean isLoginAttempt(ServletRequest request, ServletResponse response) {
        // 判断request是否包含 Authorization 字段
        String auth = getAuthzHeader(request);
        return auth != null && !"".equals(auth);
    }

    @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        if (isLoginAttempt(request,response)) {
            // executeLogin 进入登录逻辑
            // 从request请求头获取 Authorization 字段
            String token = getAuthzHeader(request);
            log.info("--->>>JwtFilter::isAccessAllowed拦截到认证token信息:[{}]",token);
            // 这里会提交给刚刚我们自定义的realm处理
            getSubject(request,response).login(new JwtToken(token));
        }
        // 这里返回true表示所有验证结果都能通过, 在controller中可以使用shiro注解限制是否需要登录权限
        // 设置true即允许游客访问
        // 设置false则必须携带token进行验证
        return true;
    }
}
```
### Shiro核心配置

核心配置 -> **ShiroConfig.java**
- 配置realm规则链
- 配置访问策略：url和filter
- 开启shiro注解支持

```java
import com.github.gongsir0630.shirodemo.filter.JwtFilter;
import org.apache.shiro.mgt.DefaultSessionStorageEvaluator;
import org.apache.shiro.mgt.DefaultSubjectDAO;
import org.apache.shiro.spring.LifecycleBeanPostProcessor;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.web.mgt.DefaultWebSecurityManager;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.DependsOn;

import javax.servlet.Filter;
import java.util.HashMap;
import java.util.Map;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/20 16:48
 * 你的指尖,拥有改变世界的力量
 * 描述: shiro核心配置
 */
@Configuration
public class ShiroConfig {
    /**
     * SecurityManager,安全管理器,所有与安全相关的操作都会与之进行交互;
     * 它管理着所有Subject,所有Subject都绑定到SecurityManager,与Subject的所有交互都会委托给SecurityManager
     * DefaultWebSecurityManager :
     * 会创建默认的DefaultSubjectDAO(它又会默认创建DefaultSessionStorageEvaluator)
     * 会默认创建DefaultWebSubjectFactory
     * 会默认创建ModularRealmAuthenticator
     */
    @Bean
    public DefaultWebSecurityManager securityManager(ShiroRealm shiroRealm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        // 设置realms
        securityManager.setRealms(shiroRealm.allRealm());
        // close session
        DefaultSubjectDAO defaultSubjectDAO = (DefaultSubjectDAO) securityManager.getSubjectDAO();
        DefaultSessionStorageEvaluator evaluator = (DefaultSessionStorageEvaluator) defaultSubjectDAO.getSessionStorageEvaluator();
        evaluator.setSessionStorageEnabled(Boolean.FALSE);
        defaultSubjectDAO.setSessionStorageEvaluator(evaluator);
        return securityManager;
    }

    /**
     * 配置Shiro的访问策略
     */
    @Bean
    public ShiroFilterFactoryBean filterFactoryBean(DefaultWebSecurityManager securityManager) {
        ShiroFilterFactoryBean factoryBean = new ShiroFilterFactoryBean();
        Map<String, Filter> filterMap = new HashMap<>(8);
        filterMap.put("jwt", new JwtFilter());
        factoryBean.setFilters(filterMap);
        factoryBean.setSecurityManager(securityManager);

        Map<String, String> filterRuleMap = new HashMap<>(8);
        //登陆相关api不需要被过滤器拦截
        filterRuleMap.put("/user/login/**", "anon");
        // 所有请求通过JWT Filter
        filterRuleMap.put("/**", "jwt");
        factoryBean.setFilterChainDefinitionMap(filterRuleMap);
        return factoryBean;
    }

    /**
     * 添加注解支持
     */
    @Bean
    @DependsOn("lifecycleBeanPostProcessor")
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator(){
        DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        advisorAutoProxyCreator.setProxyTargetClass(true);
        return advisorAutoProxyCreator;
    }

    /**
     * 添加注解依赖
     */
    @Bean
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    /**
     * 开启注解
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(DefaultWebSecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }
}
```
### 验证
实现UserService中的login方法 -> **UserServiceImpl.java**

```java
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.github.gongsir0630.shirodemo.mapper.UserMapper;
import com.github.gongsir0630.shirodemo.model.User;
import com.github.gongsir0630.shirodemo.service.UserService;
import com.github.gongsir0630.shirodemo.wx.model.WxAccount;
import com.github.gongsir0630.shirodemo.wx.service.WxAccountService;
import com.github.gongsir0630.shirodemo.wx.util.JwtUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.annotation.Resource;
import java.util.HashMap;
import java.util.Map;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 11:19
 * 你的指尖,拥有改变世界的力量
 * 描述:
 */
@Service
@Slf4j
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    @Resource
    private UserMapper userMapper;
    @Resource
    private JwtUtil jwtUtil;
    @Resource
    private WxAccountService wxAccountService;
    
    @Override
    public Map<String, String> login(String jsCode) {
        Map<String, String> res = new HashMap<>();
        WxAccount wxAccount = wxAccountService.login(jsCode);
        log.info("--->>>wxAccount信息:[{}]",wxAccount);
        User user = userMapper.selectById(wxAccount.getOpenId());
        if (user == null) {
            // todo: 用户不存在, 提醒用户提交注册信息
            res.put("canLogin",Boolean.FALSE.toString());
        } else {
            res.put("canLogin",Boolean.TRUE.toString());
        }
        res.put("token", jwtUtil.sign(wxAccount));
        return res;
    }
}
```
创建controller，编写测试api -> **UserController.java**

```java
import com.alibaba.fastjson.JSONObject;
import com.baomidou.mybatisplus.core.toolkit.StringUtils;
import com.github.gongsir0630.shirodemo.controller.res.CodeMsg;
import com.github.gongsir0630.shirodemo.controller.res.Result;
import com.github.gongsir0630.shirodemo.service.UserService;
import lombok.extern.slf4j.Slf4j;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.authz.annotation.RequiresAuthentication;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.annotation.Resource;
import java.util.Map;

/**
 * @author 程序员Kyle✨ GitHub: https://github.com/gongsir0630
 * @date 2021/3/23 11:12
 * 你的指尖,拥有改变世界的力量
 * 描述: 用户信息接口类,包含小程序登录注册
 */
@RestController
@Slf4j
@RequestMapping("user")
public class UserController {
    @Resource
    private UserService userService;

    /**
     * 从认证信息中获取用户Id: userId == openId
     * @return userId
     */
    private String getUserId() {
        return SecurityUtils.getSubject().getPrincipal().toString();
    }

    /**
     * 小程序用户登录接口: 通过js_code换取openId, 判断用户是否已经注册
     * @param code wx.login() 得到的code凭证
     * @return token
     */
    @PostMapping("/login")
    public ResponseEntity<Result<JSONObject>> login(String code) {
        if (StringUtils.isBlank(code)) {
            return new ResponseEntity<>(Result.fail(new CodeMsg(401,"code is empty"), null), HttpStatus.OK);
        }
        log.info("--->接收到来自小程序端的code:[{}]",code);
        // todo: 使用 code -> wxAccountService.login() -> openId,session_key
        Map<String, String> loginMap = userService.login(code);
        boolean canLogin = Boolean.parseBoolean(loginMap.get("canLogin"));
        String token = loginMap.get("token");
        JSONObject data = new JSONObject();
        data.put("token",token);
        data.put("canLogin",canLogin);
        log.info("--->>>返回认证信息:[{}]", data.toString());
        if (!canLogin) {
            // todo: 用户不存在,提示用户注册
            return new ResponseEntity<>(Result.fail(CodeMsg.NO_USER,data),HttpStatus.OK);
        }
        return new ResponseEntity<>(Result.success(data),HttpStatus.OK);
    }

    /**
     * 使用 RequiresAuthentication 注解, 需要验证才能访问
     * @return userId
     */
    @GetMapping("/hello")
    @RequiresAuthentication
    public ResponseEntity<Result<JSONObject>> requireAuth() {
        JSONObject data = new JSONObject();
        data.put("hello",getUserId());
        return new ResponseEntity<>(Result.success(data),HttpStatus.OK);
    }
}
```
编写小程序测试代码获取`code`：

```js
wx.login({
   timeout: 3000,
   success: (res) => {
   console.log(res);
  }
})
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ea0c2f0c10e40bb9d898198fd88d731~tplv-k3u1fbpfcp-watermark.image)

启动 [`wx-java-miniapp`](https://github.com/wx-java-miniapp)项目：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef5a62b16daa4687a1924d599037db3e~tplv-k3u1fbpfcp-watermark.image)

启动`shiro-jwt-demo`项目：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2a4d81669bc4813845add1d598674a0~tplv-k3u1fbpfcp-watermark.image)

Postman测试认证：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f23e79dc4fb4494951768161c67b2f6~tplv-k3u1fbpfcp-watermark.image)

携带token访问：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70eb9247eedc431d803bfc61484f2ee1~tplv-k3u1fbpfcp-watermark.image)

## 最后
以上就是基于Shiro、JWT实现微信小程序登录完整例子的逻辑过程说明及其实现。
- 完整项目地址：https://github.com/gongsir0630/shiro-jwt-demo

