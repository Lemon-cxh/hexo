---
title: Spring MVC参数校验
date: 2020-12-23 16:33:22
keywords: SpringBoot,@Validated,@Valid
tags:
  - Spring Boot
categories:
  - Spring Boot
description: Spring MVC中使用注解校验请求参数

---

##### JSR提供的注解

|注解|描述|
|:-:|:-:|
|@AssertFalse|所注解的元素必须是 Boolean 类型，并且值为 false|
|@AssertTrue|所注解的元素必须是 Boolean 类型，并且值为 true|
|@DecimalMax|所注解的元素必须是数字，并且它的值要小于或等于给定的BigDecimalString值|
|@DecimalMin|所注解的元素必须是数字，并且它的值要大于或等于给定的BigDecimalString值|
|@Digits|所注解的元素必须是数字，并且它的值必须有指定的位数|
|@Future|所注解的元素的值必须是一个将来的日期|
|@Past|所注解的元素的值必须是一个已过去的日期|
|@Max|所注解的元素必须是数字，并且它的值要小于或等于给定的值|
|@Min|所注解的元素必须是数字，并且它的值要大于或等于给定的值|
|@NotNull|所注解元素的值必须不能为null|
|@Null|所注解元素的值必须为null|
|@Pattern|所注解的元素的值必须匹配给定的正则表达式|
|@Size|所注解的元素的值必须是String、集合或数组，并且它的长度要符合给定的范围|

##### Hibernate Validator提供的注解

|注解|描述|
|:-:|:-:|
|@NotBlank|所注解的字符串必须不能为null且长度必须大于0|
|@Email|所注解的元素必须是电子邮箱地址|
|@Length|所注解的字符串的大小必须在指定的范围内|
|@NotEmpty|所注解的字符串必须非空|
|@Range|所注解的数字必须在合适的范围内|

##### 参数校验示例

1. 添加@Validated注解在控制器中启用校验

    ```java
    @RestController
    @RequestMapping("/")
    @Validated
    public class Controller {

        @PostMapping
        public ObjectRestResponse insert(@NotNull String name) {
            // ...
        }

    }
    ```

2. 添加@Validated或@Valid注解在参数前启用校验

    ```java
    @RestController
    @RequestMapping("/")
    public class Controller {

        @PostMapping
        public ObjectRestResponse insert(@Validated User user) {
            // ...
        }

        @PostMapping
        public ObjectRestResponse insert(@Valid User user) {
            // ...
        }

    }
    public class User {
        @NotEmpty
        @Size(min=3, max=8)
        private String name;
    }
    ```

3. 通过groups对校验进行分组
    
    > @Validated没有添加groups属性时，默认校验没有分组的校验属性
    ```java
    @RestController
    @RequestMapping("/")
    public class Controller {

        @PostMapping
        public ObjectRestResponse insert(@Validated({Insert.class}) User user) {
            // ...
        }

        @PutMapping
        public ObjectRestResponse update(@Validated({Update.class}) User user) {
            // ...
        }

    }
    public class User {
        
        @NotEmpty(groups={Update.class})
        private String id;

        @NotEmpty(groups = {Insert.class, Update.class})
        @Size(min=3, max=8, groups = {Insert.class, Update.class})
        private String name;
    }
    ```

4. 嵌套对象校验

    > 嵌套对象的校验必须在对象上添加`@Valid`注解
    ```java
    @RestController
    @RequestMapping("/")
    public class Controller {

        @PostMapping
        public ObjectRestResponse insert(@Validated User user) {
            // ...
        }

        // 必须使用@Valid
        @PostMapping
        public ObjectRestResponse insert(@Valid List<User> userList) {
            // ...
        }

    }

    public class User {
        
        @NotEmpty
        private String id;

        // 必须使用@Valid
        @Valid
        private UserInfo userInfo;
    }

    public class UserInfo {
        
        @NotEmpty
        @Size(min=3, max=8)
        private String name;
    }
    ```

5. 使用全局异常处理返回错误信息

    ```java
    @ControllerAdvice
    @ResponseBody
    public class GlobalExceptionHandler {

        @ExceptionHandler(MethodArgumentNotValidException.class)
        public ObjectRestResponse handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
            return new ObjectRestResponse().rel(false)
                .message(ex.getBindingResult().getFieldError().getDefaultMessage());
        }

        @ExceptionHandler(BindException.class)
        public ObjectRestResponse handleBindException(BindException ex) {
            return new ObjectRestResponse().rel(false)
                .message(ex.getBindingResult().getFieldError().getDefaultMessage());
        }
    }
    ```

6. 使用BindingResult返回错误信息
    
    ```java
    @PostMapping
    public ObjectRestResponse insert(@Validated User user, BindingResult result) {
        if (result.hasErrors()) {
            return new ObjectRestResponse().rel(false)
                .message(result.getFieldError().getDefaultMessage());
        }
        // ...
    }
    ```
