---
title: 遇到的问题
date: 2020-04-10 14:14:19
keywords: problems,问题
tags:
- problems
description: 开发过程中遇到的问题,以及处理方式
---
##### Spring Boot

1. 服务启动后即关闭，没有错误日志

     ```java
    public static void main(String[] args) {
        try {
            SpringApplication.run(TestBootstrap.class, args);
        } catch (Exception e) {
            log.error("error", e);
        }
    }
    ```

2. 服务部署启动

##### Spring Cloud Gateway

1. Invalid character '[' for QUERY_PARAM in "match[]"

    ```vim
    java.lang.IllegalArgumentException: Invalid character '[' for QUERY_PARAM in "match[]"
    ```
    Spring Cloud Gateway与Webflux的兼容性的问题，项目的Gateway版本2.0.0.RELEASE，去除`spring-boot-starter-webflux`依赖解决可以升级Gateway版本。[issues](https://github.com/spring-cloud/spring-cloud-gateway/issues/462)

2.  unexpected message type: DefaultHttpRequest

    ```vim
    io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: unexpected message type: DefaultHttpRequest
    ```
    Spring Cloud Gateway以及Spring Boot版本2.0.1.RELEASE更新为2.0.4.RELEASE。[issues](https://github.com/reactor/reactor-netty/issues/177)

##### Spring MVC
   
1. Spring MVC 项目，请求返回404
    Spring Boot 使用惯了，接到 Spring MVC 的反而懵了，在 Controller 中定义好了接口，但是访问就404。
        想到 Spring MVC 的 `org.springframework.web.servlet.DispatcherServlet`,于是在 doDispatch() 方法中加上断点进行调试。发现 this.getHandler() 返回了 Null，即 HandlerMapping 为 Null。因为 Controller 中使用了 @RequestMapping 注解，而且项目是使用 XML 的配置方式，所以需要手动配置 RequestMappingHandlerMapping。(如果 Spring MVC 是旧版本也需要手动配置 HandlerAdapter，即对应的RequestMappingHandlerAdapter)
    ```xml
        <!-- 配置HandlerMapping -->
        <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping" />
        <!-- 配置HandlerAdapter -->
        <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter" />
    ```


##### Nginx

1. upstream sent no valid HTTP/1.0 header while reading response header from upstream

    Nginx报错，然后设置了Nginx和业务服务配置
    ```conf
    proxy_http_version 1.1;
    proxy_cache_bypass      $http_upgrade;

    proxy_set_header Upgrade                $http_upgrade;
    proxy_set_header Connection "";
    proxy_set_header Host                   $host;
    proxy_set_header X-Real-IP              $remote_addr;
    proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto      $scheme;
    proxy_set_header X-Forwarded-Host       $host;
    proxy_set_header X-Forwarded-Port       $server_port;
    ```
    ```yml
    server:
        use-forward-headers: true
        tomcat:
            remote-ip-header: x-forwarded-for
            protocol-header: x-forwarded-proto
            port-header: X-Forwarded-Port
    ```
        
    然后依旧报错，此时把服务停止运行再次请求，然而正常访问，说明Spring Gateway和Nginx之间没有问题，便排查业务服务。
        在`HandlerInterceptorAdapter`处理拦截器种发现了这么一段代码`response.sendError`。
    此方法使用指定的状态码发送一个错误响应至客户端。服务器默认会创建一个HTML格式的服务错误页面作为响应结果，其中包含参数msg指定的文本信息，这个HTML页面的内容类型为`text/html`，保留cookies和其他未修改的响应头信息。
        最后去掉此行代码，替换为抛出全局异常，得以解决。
    ```java
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        ***
        response.sendError(***);
        ***
    }
    ```

2. upstream server temporarily disabled while reading response header from upstream
3. no live upstreams while connecting to upstream

    ```conf
    upstream zxjl {
        least_conn;
        server 192.168.1.1:8765 max_fails=3 fail_timeout=30s;
        server 192.168.1.2:8765 max_fails=3 fail_timeout=30s;
        keepalive 256;
    }
    ```

    > max_fails 默认值为1, fail_timeout 默认值为 10 秒。

    fail 失败会返回:
    - upstream timed out (110: Connection timed out) while reading response header from upstream
    - connect() failed (111: Connection refused) while connecting to upstream

##### 服务器

1. Error: Network Error
    请求时不时提示net:ERR_CONNECTION_TIMED_OUT以及Error: Network Error。
    ```conf
    # /etc/sysctl.conf
    net.ipv4.tcp_tw_recycle = 0
    ```

##### MySQL

1. Slave_SQL_Running: No

    因为设置主从同步时忘记排除mysql库，导致主从同步故障。
    ```bin
    slave stop;
    # 跳过一个复制错误
    set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
    slave start;
    ```

##### Redis

1. 分布式锁的错误实现
    听到同事说到业务加了Redis实现的分布式锁，但是没有偶尔没有生效，会重复添加两次数据。查看代码发现大致实现逻辑如下：
    ```java
    if (Boolean.TRUE.equals(redisTemplate.hasKey(key))) {
        throw new BizException("请勿重复提交");
        return;
    }
    redisTemplate.opsForValue().set(key, "1", 3, TimeUnit.SECONDS);
    ```
    可以使用 hasKey() 方法来检查 Redis 中是否存在指定的键。但是，在实现分布式锁时，使用 hasKey() 方法并不能保证原子性操作。

    考虑以下情况：线程 A 和线程 B 同时调用 hasKey() 方法检查 Redis 是否存在锁键。假设两个线程都发现键不存在，然后都尝试设置锁键。这样就会导致多个线程同时获取到了锁，破坏了锁的互斥性。

    为了保证原子性操作，需要使用 Redis 的原子性指令，如 setIfAbsent() 命令。该命令在键不存在时设置键的值，并且保证了原子性操作

##### Jackson

1. missing type id property '@Clazz'
   
    在接收请求外部接口接收响应值时，报错：
    ```
    Error while extracting response for type [*.**] and content type [application/json], feign.InvocationContext.decode(InvocationContext.java)
    ```
    报错信息在`@RestControllerAdvice`的全局拦截异常中的`@EeceptionHandler({FeignException.class})`方法中，于是打断点查看更详细的报错信息，报错为：
    ```
    Could not resolve subtype of [simple type, class *.**]: missing type id property '@Clazz'
    ```
    问题是 Jackson 在反序列化时无法找到所需的类型标识属性。在无法修改外部请求代码的情况下时，只能修改本地代码，去除对应的反序列化配置
    ```java
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer json2ObjectMapperBuilderCustomizer() {
        return builder -> builder.mixIn(IPage.class, IPageMixin.class);
    }

    @JsonIgnoreProperties(ignoreUnknown = true)
    @JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "@Clazz")
    interface IPageMixin {}
    ```

##### hibernate-validator

1. 对于@NotNull的返回信息为：**不能为null
    在resources下创建对应的配置文件，更改默认信息即可。参考：[hibernate-validator](https://github.com/hibernate/hibernate-validator/blob/main/engine/src/main/resources/org/hibernate/validator/ValidationMessages_zh_CN.properties)

##### 其他

1. 项目前端请求的时候会弹出登录的弹框，接口响应401，请求头中的`Authorization`的值是以`Basic`开头

    ![alt text](login.png)
 
   项目使用JWT鉴权，在请求头中使用`Authorization`传递Token，值应该以`Bearer`开头。想到Swagger的鉴权是使用`Basic`，所以这块应该是携带了Swagger的请求头。所以需要清除Swagger的用户名和密码，在 Chrome 中，我们只要在 URL 前面加上`user@`即可强制浏览器刷新它的缓存。

   [HTTP Basic Auth 是怎么样工作的](https://cjting.me/2018/03/31/how-http-basic-auth-work/)