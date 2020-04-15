---
title: SpringBoot字符串参数去除头尾空格
date: 2020-01-05 15:29:26
categories: 
- Spring Boot
tags:
- Spring Boot
description: SpringBoot对请求参数和请求体JSON中的字符串去除头尾空格
---
1. ##### URL和FORM表单中的参数
    ```java
    @ControllerAdvice
    public class ControllerStringParamTrimConfig {

        @InitBinder
        public void initBinder(WebDataBinder binder) {
            // 构造方法中 boolean 参数含义为如果是空白字符串,是否转换为null
            // 即如果为true,那么 " " 会被转换为 null,否者为 ""
            binder.registerCustomEditor(String.class, new StringTrimmerEditor(true));
        }
    }
    ```

2. ##### RequestBody中JSON对象参数
    ```java
    @Configuration
    public class JacksonConfiguration {

        @Bean
        public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
            return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder
                    .deserializerByType(String.class, new StdScalarDeserializer<String>(String.class) {
                        @Override
                        public String deserialize(JsonParser jsonParser, DeserializationContext ctx)
                                throws IOException {
                            return StringUtils.trimWhitespace(jsonParser.getValueAsString());
                        }
                    });
        }
    }
    ```
3. ##### 原文链接
    > [ghthou：Spring MVC 配置接收 String 参数时自动去除前后空格](https://ghthou.github.io/2018/10/04/Spring-MVC-%E9%85%8D%E7%BD%AE%E6%8E%A5%E6%94%B6-String-%E5%8F%82%E6%95%B0%E6%97%B6%E8%87%AA%E5%8A%A8%E5%8E%BB%E9%99%A4%E5%89%8D%E5%90%8E%E7%A9%BA%E6%A0%BC/)