---
title: Spring Cloud 版本升级为Greenwich，Feign 遇到的问题
date: 2020-07-29 16:51:37
keywords: Feign,SpringCloud
tags:
- Spring Cloud Feign
description: Feign来调用Get请求,参数为POJO类时需要添加@SpringQueryMap注解，因此Spring Cloud版本从Finchley升级到Greenwich，之后所遇到的问题
---

> Feign 来调用 Get 请求,参数为 POJO 类时需要添加 `@SpringQueryMap` 注解，因此 Spring Cloud 版本从 Finchley 升级到 Greenwich

1. ### 服务启动失败，报错已经定义了具有该名称的Bean

    #### 错误信息

    ```
    The bean 'FeignClientSpecification' could not be registered. A bean with that name has already been defined and overriding is disabled.
    ```

    #### 解决办法

    在`@FeignClient`注解上添加`contextId`属性，配置了contextId就会用contextId，如果没有配置就会去value然后是name最后是serviceId。默认都没有配置，当出现一个服务有多个Feign Client的时候就会报错了

    #### 参考

    - [那天晚上和@FeignClient注解的深度交流](https://juejin.im/post/5e13e8116fb9a0481e2796a3#heading-5)

2. ### 没有POJO类的父类的属性

    #### 解决办法

    添加如下配置

    ```java
    @Bean
    public Feign.Builder feignBuilder() {
        return Feign.builder()
                .queryMapEncoder(new BeanQueryMapEncoder())
                .retryer(Retryer.NEVER_RETRY);
    }
    ```

    #### 参考

    - [Feign发送get请求使用对象传参问题，@SpringQueryMap解析传参对象父类属性解决方案](https://blog.csdn.net/li295214001/article/details/90410945)

3. ### 返回结果为JSON，但是ContentType不符

    #### 错误信息

    ```
    Could not extract response: no suitable HttpMessageConverter found for response type [class com.alibaba.fastjson.JSONObject] and content type [text/html;charset=utf-8]
    ```

    #### 解决办法

    添加如下配置

    ```java
    @Bean
    public Decoder feignDecoder(){
        return new SpringDecoder(() -> new HttpMessageConverters(new TextMessageConverter()));
    }

    public static class TextMessageConverter extends MappingJackson2HttpMessageConverter {
        public TextMessageConverter(){
            List<MediaType> mediaTypes = new ArrayList<>();
            mediaTypes.add(MediaType.TEXT_PLAIN);
            mediaTypes.add(MediaType.TEXT_HTML);
            setSupportedMediaTypes(mediaTypes);
        }
    }
    ```

4. ### 开发时查看Feign请求日志

    application配置文件
    ```yml
    logging:
        level:
            root: DEBUG
    feign:
        client:
            config:
                default:
                    loggerLevel: basic
    ```