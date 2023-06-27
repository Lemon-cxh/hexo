---
title: 自定义一些公用的 SpringBoot Starter
tags:
  - Spring Boot
keywords: SpringBoot
categories:
  - Spring Boot
description: 自定义 Cahce、Feign、Log、Web、等 Spring Boot Starter
date: 2022-08-02 15:07:59
---


- #### cache: 缓存模块

  > spring-boot-starter-cache + redisson + caffeine + aop

  1. 开启缓存 `@EnableCaching`
  2. 根据配置构建单机、集群、哨兵模式的 RedissonClient，配置codec(JsonJacksonCodec)
  3. 创建自定义 CacheService，二次封装redisson的基础操作
  4. 构建自定义的 RedisTemplate、StringRedisTemplate
  5. 根据配置构建 Redisson、Caffeine、Redisson + Caffeine 的 CacheManager 缓存管理器
    - RedissonSpringCacheManager: 
      1. 实现 KeyGenerator 自定义 key 生成规则
      2. 使用上述配置的 RedissonClient
      3. 实现 CacheManager 自定义缓存管理器，设置自定义缓存命名空间列表的过期时间
    - CaffeineCacheManager:
      1. 实现 KeyGenerator 自定义 key 生成规则
      2. 实现 CacheManager 自定义缓存管理器，设置自定义缓存命名空间列表的过期时间
    - L2SpringCacheManage: 自实现 Redisson + Caffeine 的二级缓存
  6. 注入分布式锁切面，拦截 @DistributeLock 注解的方法
    - 依赖注入上述配置的 RedissonClient
    - 遍历 @DistributeLock 的 Spel 表达式，添加到 set 中
    - 遍历列表尝试加锁，失败则抛出异常，finally 时对列表解锁
  7. 将 GenericJackson2JsonRedisSerializer 注入容器，替换 SpringSessionDefaultRedisSerializer 处理 SpringSession 的 Redis 序列化

- #### feign: Feign配置

  1. 构建请求拦截器 `RequestInterceptor`, 设置请求头参数 Authorization，尝试从请求头、当前线程变量中获取
  2. 配置 `OkHttpClient`
  3. 创建 `ErrorDecoder.Default` 实现, 自定义错误异常
  4. 配置 FeignExceptionHandler 捕获 @FeignRestController 的异常
  5. 配置 `feign.Logger` 实现, 自定义异常解析
  6. 定义切面，拦截 @FeignSsoClientAuth 注解的方法设置线程变量 Token
  7. 配置 Caffeine 用于缓存 Token

- #### job: XXL-Job

  > [分布式任务调度平台XXL-JOB](https://www.xuxueli.com/xxl-job/#2.4%20%E9%85%8D%E7%BD%AE%E9%83%A8%E7%BD%B2%E2%80%9C%E6%89%A7%E8%A1%8C%E5%99%A8%E9%A1%B9%E7%9B%AE%E2%80%9D)

  - 注入配置文件属性，构建注入 XxlJobSpringExecutor

- #### log: log相关

  1. 配置 AsyncTaskExecutor 线程池
  2. 定义日志切面，拦截 @ApiOperation 注解的方法，设置环绕、异常切点，构建Log对象，调用 AccessLogAppender 接口记录日志
  3. 根据配置构建 Console、File、Elasticsearch 的 AccessLogAppender 实现类

- #### oss: oss存储

  1. 根据配置构建相应的 OssStorageService 实现类

- #### rdb: MybatisPlus配置

  1. 启用事务管理器 `@EnableTransactionManagement`
  2. 配置插件 `MybatisPlusInterceptor`，添加租户、分页、乐观锁、防止全表更新与删除、数据权限插件，构建自动填充处理器 `MyMetaObjectHandler`

- #### sso: SpringSecurity

  1. 启用 Security `@EnableWebSecurity`、`@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)`
  
    - prePostEnabled: 开启 prePostEnabled 相关的注解
      - @PreAuthorize 在方法调用之前,基于表达式的计算结果来限制对方法的访问
      - @PostAuthorize 允许方法调用,但是如果表达式计算结果为false,将抛出一个安全性异常
      - @PostFilter 允许方法调用,但必须按照表达式来过滤方法的结果
      - @PreFilter 允许方法调用,但必须在进入方法之前过滤输入值
    - securedEnabled: 开启 @Secured 注解过滤权限，只有满足角色的用户才能访问被注解的方法
    - jsr250Enabled: 开启 JSR-205 注解
      - @DenyAll 拒绝所有的访问
      - @PermitAll 运行所有访问
      - @RolesAllowed({"USER","ADMIN"}) 只有满足角色的用户才能访问被注解的方法
  
  2. 实现 `ResourceServerConfigurerAdapter`，自定义 accessDeniedHandler、authenticationEntryPoint 的返回格式
  
  3. 实现 `WebSecurityConfigurerAdapter`，自定义安全访问策略

    - void configure(AuthenticationManagerBuilder auth)：配置认证管理器 AuthenticationManager
    - void configure(WebSecurity web)：配置 WebSecurity，使用 ignoring() 方法用来忽略 Spring Security 对静态资源的控制(例如: swagger)
    - void configure(HttpSecurity http)：配置 HttpSecurity，自定义安全访问策略
  
  4. 实现 `HandlerInterceptor`，提取 Authentication 信息到线程变量中
  
  5. 实现 `WebMvcConfigurer`，添加自定义的拦截器拦截 /** 路径
  
  6. 实现 `Filter` 自定义对访问的权限过滤，是否有次资源服务器的权限，具体的接口权限等判断


- #### web: web配置

  1. 实现 WebMvcConfigurer，addResourceHandlers 方法定义静态资源处理器
  2. 配置内嵌 Tomcat 允许特殊字符 ConfigurableServletWebServerFactory.TomcatConnectorCustomizer
  3. 配置 MappingJackson2HttpMessageConverter，设置 ObjectMapper 的 JavaTimeModule，添加 LocalTime、LocalDate、LocalDateTime 相关的序列化、反序列化，设置  DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES 属性为 false
  4. 自定义了RequestMappingHandlerMapping、RequestCondition，根据@ApiVersion来匹配对应的Handler
  5. 配置 Converter 自定义 String 转 Date
  6. 配置 GlobalExceptionHandler 全局异常拦截处理器
  7. 定义切面，拦截 @NoRepeat 注解方法，实现防止重复请求(移到cache模块合适点?)
  8. CurrentUserService：保存用户信息的 ThreadLocal

- #### swagger: swagger文档

  1. 定义配置属性类，读取 application 文件
  2. 定义配置类，添加 @EnableSwagger2 注解，注入配置属性用以配置 Swagger


- #### 自定义注解

  - @AdviceSpringCloudApplication:
    继承@SpringBootApplication、@EnableDiscoveryClient(服务注册)、@EnableFeignClients(启用Feign)、@EnableAsync(启用异步)、@EnableScheduling(启用计划任务)、@EnableCircuitBreaker(熔断、限流)、@ServletComponentScan(使@WebServlet、@WebFilter、@WebListener自动注册到容器)、@Documented、@Inherited
  - @FeignRestController
    用于捕获 Feign Controller 的异常
  - @AdviceRestController、 @AdviceController
    用于捕获 Controller 的异常
  - @DistributeLock
    分布式锁
  - @NoRepeat
    防止重复请求
  - @FeignSsoClientAuth
    用于添加 Feign 请求所需的 Token
  - @NoAppendAccessLog
    不记录访问日志
  - @SsoPermission
    自定义权限拦截
