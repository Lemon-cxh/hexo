---
title: 'Mybatis拦截器之数据加密解密'
date: 2019-11-12 16:18:07
categories: 
- Mybatis
tags:
- Mybatis
description: 我们要实现数据加密，进入数据库的字段不能是真实的数据，但是返回来的数据要真实可用，所以我们需要实现自定义的拦截器
---
1. #### 拦截器介绍
    
    MyBatis 允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：
    - `执行:`Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
    - `请求参数处理:`ParameterHandler (getParameterObject, setParameters)
    - `返回结果集处理:`ResultSetHandler (handleResultSets, handleOutputParameters)
    - `SQL语句构建:`StatementHandler (prepare, parameterize, batch, update, query)

    这些类中方法的细节可以通过查看每个方法的签名来发现，或者直接查看 MyBatis 发行包中的源代码。 如果你想做的不仅仅是监控方法的调用，那么你最好相当了解要重写的方法的行为。 因为如果在试图修改或重写已有方法的行为的时候，你很可能在破坏 MyBatis 的核心模块。 这些都是更低层的类和方法，所以使用插件的时候要特别当心。

    通过 MyBatis 提供的强大机制，使用插件是非常简单的，只需实现 Interceptor 接口，并指定想要拦截的方法签名即可。具体可以参考[官方文档](http://www.spring4all.com/article/15081)。

2. #### 拦截器的使用
    我们要实现数据加密，进入数据库的字段不能是真实的数据，但是返回来的数据要真实可用，所以我们需要针对 Parameter 和 ResultSet 两种类型处理，同时为了更灵活的使用，我们需要自定义注解。

    1. ##### 自定义注解
        我们可以在我们要处理的实体和实体中的字段加上需要的注解
        ```java
        /**
         * 需要加解密的类注解
         */
        @Documented
        @Inherited
        @Target({ ElementType.TYPE })
        @Retention(RetentionPolicy.RUNTIME)
        public @interface EncryptDecryptClass {}
        ```
        ```java
        /**
         * 加密字段注解
         */
        @Documented
        @Inherited
        @Target({ ElementType.FIELD })
        @Retention(RetentionPolicy.RUNTIME)
        public @interface EncryptDecryptField {

            /**
            * 字段是否使用,分割符号拼接而成(SQL查询使用GROUP_CONCAT)
            * @return boolean
            */
            boolean split() default false;
        }
        ```

    2. ##### 自定义参数处理拦截器
        参考官网，通过 @Intercepts 和 @Signature 的联合使用，指定 ParameterHandler.class 类型，同时通过 @Component 注解注入到容器中，即可在设置参数的时候进行拦截，根据 Field 的各种类型自定义加密解密算法
        ```java
        @Intercepts({
            @Signature(type = ParameterHandler.class, method = "setParameters", args = PreparedStatement.class)
        })
        @Component
        public class ParammeterInterceptor  implements Interceptor {

            private Properties properties = new Properties();

            @Override
            public Object intercept(Invocation invocation) throws Throwable {
                if (!(invocation.getTarget() instanceof ParameterHandler)) {
                    return invocation.proceed();
                }
                ParameterHandler parameterHandler = (ParameterHandler) invocation.getTarget();
                Object parameterObject = parameterHandler.getParameterObject();
                if (Objects.isNull(parameterObject)) {
                    return invocation.proceed();
                }
                Class<?> parameterObjectClass = parameterObject.getClass();
                EncryptDecryptClass encryptDecryptClass = parameterObjectClass.getAnnotation(EncryptDecryptClass.class);
                if (Objects.nonNull(encryptDecryptClass)){
                    EncryptDecryptUtils.encrypt(EncryptDecryptUtils.getFields(parameterObjectClass), parameterObject);
                }
                return invocation.proceed();
            }

            @Override
            public Object plugin(Object o) {
                return Plugin.wrap(o, this);
            }

            @Override
            public void setProperties(Properties properties) {
                this.properties = properties;
            }
        }
        ```

    3. ##### 结果集拦截器
        与参数拦截器基本一样, 只不过类型指定为 ResultSetHandler.class
        ```java
        @Intercepts({
            @Signature(type = ResultSetHandler.class, method = "handleResultSets", args={Statement.class})
        })
        @Component
        public class ResultInterceptor implements Interceptor {

            private Properties properties = new Properties();

            @Override
            public Object intercept(Invocation invocation) throws Throwable {
                Object result = invocation.proceed();
                if (Objects.isNull(result)){
                    return null;
                }
                if (result instanceof List) {
                    EncryptDecryptUtils.fieldListDecrypt(result);
                }else if(EncryptDecryptUtils.notToDecrypt(result)) {
                    EncryptDecryptUtils.decrypt(EncryptDecryptUtils.getFields(result.getClass()), result);
                }
                return result;
            }

            @Override
            public Object plugin(Object target) {
                return Plugin.wrap(target, this);
            }

            @Override
            public void setProperties(Properties properties) {
                this.properties = properties;
            }
        }
        ```

    4. ##### 加密解密工具类
        ```java
        public class EncryptDecryptUtils {

            /**
            * 多field加密方法
            *
            * @param declaredFields 字段
            * @param parameterObject 类
            * @return T
            * @throws IllegalAccessException IllegalAccessException
            */
            public static <T> void encrypt(Field[] declaredFields, T parameterObject) throws IllegalAccessException {
                for (Field field : declaredFields) {
                    encrypt(field, parameterObject);
                }
            }

            /**
            * 多个field解密方法
            *
            *
            * @param declaredFields 解密字段
            * @param result 结果
            * @throws IllegalAccessException IllegalAccessException
            */
            public static void decrypt(Field[] declaredFields, Object result) throws IllegalAccessException {
                for (Field field : declaredFields) {
                    decrypt(field, result);
                }
            }

            /**
            * 单个field加密方法
            *
            * @param field 字段
            * @param parameterObject 类
            * @return T
            * @throws IllegalAccessException IllegalAccessException
            */
            private static <T> void encrypt(Field field, T parameterObject) throws IllegalAccessException {
                field.setAccessible(true);
                Object object = field.get(parameterObject);
                if (object instanceof String) {
                    field.set(parameterObject, AESUtil.encrypt((String)object));
                }
            }

            /**
            * 单个field解密方法
            *
            * @param field 字段
            * @param result 结果
            * @throws IllegalAccessException IllegalAccessException
            */
            private static void decrypt(Field field, Object result) throws IllegalAccessException {
                fieldObjectDecrypt(field, result);
                field.setAccessible(true);
                Object object = field.get(result);
                if (Objects.isNull(object)) {
                    return;
                }
                if (object instanceof String) {
                    EncryptDecryptField decryptField = field.getAnnotation(EncryptDecryptField.class);
                    if (!decryptField.split()) {
                        field.set(result, AESUtil.decrypt((String)object));
                        return;
                    }
                    String[] strings = ((String) object).split(",");
                    StringBuilder stringBuilder = new StringBuilder();
                    for (int i = 0, length = strings.length - 1; i < length; i++) {
                        stringBuilder.append(AESUtil.decrypt(strings[i])).append(",");
                    }
                    stringBuilder.append(AESUtil.decrypt(strings[strings.length - 1]));
                    field.set(result, stringBuilder.toString());
                }
            }

            /**
            * 是否解密
            * @param object 对象
            * @return 是否解密
            */
            public static boolean notToDecrypt(Object object){
                Class<?> objectClass = object.getClass();
                EncryptDecryptClass encryptDecryptClass = objectClass.getAnnotation(EncryptDecryptClass.class);
                return Objects.isNull(encryptDecryptClass);
            }

            /**
            * 获取有@EncryptDecryptField注解的字段
            * @param objectClass 类
            * @return 字段
            */
            public static Field[] getFields(Class<?> objectClass) {
                Field[] fields = Arrays.stream(objectClass.getDeclaredFields())
                        .filter(field -> Objects.nonNull(field.getAnnotation(EncryptDecryptField.class)))
                        .toArray(Field[]::new);
                Class<?> superClass = objectClass.getSuperclass();
                if (Object.class == superClass) {
                    return fields;
                }
                return ArrayUtils.addAll(fields, Arrays.stream(superClass.getDeclaredFields())
                        .filter(field -> Objects.nonNull(field.getAnnotation(EncryptDecryptField.class)))
                        .toArray(Field[]::new));
            }

            /**
            * 字段对象的解密
            * @param field 字段
            * @param result 数据
            * @throws IllegalAccessException IllegalAccessException
            */
            private static void fieldObjectDecrypt(Field field, Object result) throws IllegalAccessException {
                field.setAccessible(true);
                Object object = field.get(result);
                if (object instanceof List) {
                    fieldListDecrypt(object);
                    return;
                }
                Class<?> objectClass = field.getType();
                EncryptDecryptClass encryptDecryptClass = objectClass.getAnnotation(EncryptDecryptClass.class);
                if (Objects.nonNull(encryptDecryptClass)) {
                    decrypt(getFields(objectClass), field.get(result));
                }
            }

            /**
            * 字段集合的解密
            * @param object 集合数据
            * @throws IllegalAccessException IllegalAccessException
            */
            public static void fieldListDecrypt(Object object) throws IllegalAccessException {
                ArrayList resultList = (ArrayList) object;
                if (resultList.isEmpty() || EncryptDecryptUtils.notToDecrypt(resultList.get(0))){
                    return;
                }
                Field[] declaredFields = EncryptDecryptUtils.getFields(resultList.get(0).getClass());
                if (ArrayUtils.isEmpty(declaredFields)) {
                    return;
                }
                for (Object o : resultList) {
                    decrypt(declaredFields, o);
                }
            }

            private EncryptDecryptUtils() {}
        }
        ```
        
    > 数据库同样有AES加解密函数：`AES_ENCRYPT(str,key_str)`以及`AES_DECRYPT(crypt_str,key_str)`。可以执行`SELECT @@block_encryption_mode;` 查看加密模式,修改为256位:`SET @@block_encryption_mode = 'aes-256-ecb';`

3. #### 原文链接
    > ### [Fraser Yu：Mybatis拦截器之数据加密解密](http://www.spring4all.com/article/15081)