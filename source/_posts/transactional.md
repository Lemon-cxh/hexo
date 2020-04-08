---
title: 事务配置@Transactional
tags:
  - SpringBoot
categories:
  - SpringBoot
description: SpringBoot声明式事务的使用
date: 2020-04-07 11:46:07
---
1. ##### @Transactional的配置

    ```java
    public @interface Transactional {
        // 通过 bean name 指定事务管理器
        @AliasFor("transactionManager")
        String value() default "";

        // 同 value 属性
        @AliasFor("value")
        String transactionManager() default "";

        // 指定传播行为
        Propagation propagation() default Propagation.REQUIRED;

        // 指定隔离级别
        Isolation isolation() default Isolation.DEFAULT;

        // 指定超时时间(单位秒)
        int timeout() default -1;

        // 是否只读事务
        boolean readOnly() default false;

        // 方法在发生指定异常时回滚，默认是所有异常都回滚
        Class<? extends Throwable>[] rollbackFor() default {};

        // 方法在发生指定异常名称时回滚，默认是所有异常都回滚
        String[] rollbackForClassName() default {};

        // 方法在发生指定异常时不回滚，默认是所有异常都回滚
        Class<? extends Throwable>[] noRollbackFor() default {};

        // 方法在发生指定异常名称时不回滚，默认是所有异常都回滚
        String[] noRollbackForClassName() default {};
    }
    ```

2. ##### Isolation隔离级别

    - `DEFAULT`——数据库默认隔离级别，MySQL是可重复读，Oracle是已提交读
    - `READ_UNCOMMITTED`——未提交读：事务读取其他事务未提交的数据
    - `READ_COMMITTED`——已提交读：事务读取最新的其他事务已提交的数据
    - `REPEATABLE_READ`——可重复读：事务读取该事务开始的时间点的其他事务已提交的数据
    - `SERIALIZABLE`——可序列化：所有事务只能一个接一个串行执行，不能并发

    ---

    可能发生的问题：
    - `脏读`——事务T1读取了已被事务T2更新但还没有提交的数据，若T2回滚，T1读取到的就是无效内容
    - `不可重复读`——事务T1读取一个数据两次，在第一次和第二次之间，T2更新了该数据，导致T1第二次读取到的数据不同
    - `幻读`——事务T1读取若干行数据两次，在第一次和第二次之间，T2新增或删除该数据集，导致T1第二次读取到的数据条数不同

    > 幻读与不可重复读之间的区别是幻读强调的是新增或删除，而不可重复读强调的是修改。

    ---

    |类型|脏读|不可重复读|幻读|
    |:--:|:-:|:-:|:-:|
    |未提交读|√|√|√|
    |已提交读|×|√|√|
    |可重复读|×|×|√|
    |可序列化|×|×|×|

3. ##### Propagation传播行为

    - ***`REQUIRED`***——默认传播行为，需要事务，如果当前存在事务，就沿用当前事务，否则新建一个事务运行子方法
    - `SUPPORTS`——支持事务，如果当前存在事务，就沿用当前事务，如果不存在，则继续采用无事务的方式运行子方法
    - `MANDATORY`——必须使用事务，如果当前没有事务，则会抛出异常，如果存在当前事务 就沿用当前事务
    - ***`REQUIRES_NEW`***——无论当前事务是否存在，都会创建新事务运行方法
    - `NOT_SUPPORTED`——不支持事务，当前存在事务时，将挂起事务，运行方法
    - `NEVER`——不支持事务，如果当前方法存在事务，则抛出异常，否则继续使用无事务机制运行
    - ***`NESTED`***——在当前方法调用子方法时，如果子方法发生异常，只因滚子方法执行过的SQL，而不回滚当前方法的事务
