---
title: MyBatis的缓存机制
tags:
  - MyBatis
categories:
  - MyBatis
description: MyBatis一级缓存与二级缓存的介绍，执行流程
date: 2021-03-13 17:21:04
---


#### 一级缓存

##### 一级缓存介绍
Mybatis默认开启一级缓存。一级缓存是SqlSession级别的缓存，每个SqlSession持有一个Executor，每个Executor中有一个LocalCache。当发起查询时，MyBatis根据当前执行的语句生成MappedStatement，在Local Cache进行查询，如果缓存命中的话，直接返回结果给，如果缓存没有命中的话，查询数据库，结果写入Local Cache，最后返回结果。
- SqlSession： 对外提供了用户和数据库之间交互需要的所有方法，隐藏了底层的细节
- Executor： SqlSession向用户提供操作数据库的方法，但和数据库操作有关的职责都会委托给Executor
- Cache： MyBatis中的Cache接口，提供了和缓存相关的最基本的操作
  
![L1-flow](L1-flow.png)

##### 一级缓存流程

1. 首先初始化SqlSession，在初始化SqlSesion时，会创建一个全新的Executor。
2. SqlSession创建完毕后，根据Statment的不同类型，会进入SqlSession的不同方法中，如果是Select语句，会执行SqlSession的selectList方法。SqlSession把具体的查询职责委托给了Executor。如果只开启了一级缓存的话，首先会进入Executor的query方法。
3. Executor的query方法根据CacheKey在LocalCache的HashMap中查询数据。如果查不到的话，就从数据库查，并对localcache进行写入。在query方法执行的最后，会判断一级缓存级别是否是STATEMENT级别，如果是的话，就清空缓存，这也就是STATEMENT级别的一级缓存无法共享LocalCache的原因。

    > CacheKey由MappedStatement的Id、SQL的offset、SQL的limit、SQL本身以及SQL中的参数构成。如果五个值相同，即可以认为是相同的SQL。

4. SqlSession的insert方法和delete方法，都会统一走update的流程，update方法也是委托给了Executor执行，每次执行update前都会清空LocalCache。

![cache-flow](cache-flow.png)

##### 一级缓存配置
MyBatis的一级缓存最大范围是SqlSession内部，有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement。
```yml
mybatis:
  configuration:
    localCacheScope: STATEMENT // 默认SESSION,
```
> 有多个SqlSession或者分布式的环境下，数据库写操作会引起脏数据，建议设定缓存级别为Statement。

#### 二级缓存

##### 二级缓存介绍
一级缓存中，其最大的共享范围就是一个SqlSession内部，如果多个SqlSession之间需要共享缓存，则需要使用到二级缓存。开启二级缓存后，会使用CachingExecutor装饰Executor，进入一级缓存的查询流程前，先在CachingExecutor进行二级缓存的查询。
![L2-flow](L2-flow.png)

##### 二级缓存流程

二级缓存在一级缓存处理前，用CachingExecutor装饰了BaseExecutor的子类，在委托具体职责给delegate之前，实现了二级缓存的查询和写入功能。
MyBatis的CachingExecutor持有一个TransactionalCacheManager(缓存管理器)，TransactionalCacheManager中持有一个`transactionalCaches`的Map。
这个Map保存了Cache和用TransactionalCache包装后的Cache的映射关系。

```java
private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap();
```

> transactionalCaches的大小取决于MappedStatement的数量。
> TransactionalCache中的`clearOnCommit`属性为了避免多线程的脏读现象。
> TransactionalCache中的`entriesToAddOnCommit`属性用于暂存数据

- 修改：会将TransactionalCache的`clearOnCommit`属性设置为true,并将暂存的数据清空，未清空二级缓存。
- 查询：从二级缓存查询数据，最后判断`clearOnCommit`为true则直接返回null。
- commit：`clearOnCommit`为true则清除二级缓存，并将`entriesToAddOnCommit`中的数据存放到二级缓存中。


#### 二级缓存设置
- 配置文件
    ```yml
    mybatis:
    configuration:
        cache-enabled: true // 默认为true
    ```
- Mapper文件
    cache标签用于声明这个namespace使用二级缓存,或者在Mapper接口类上添加`@CacheNamespace`注解
    ```xml
    <cache/>
    ```
    cache-ref代表引用别的命名空间的Cache配置，两个命名空间的操作使用的是同一个Cache，多表查询时使用。或者在Mapper接口类上添加`@CacheNamespaceRef(StudentMapper.class)`注解
    ```xml
    <cache-ref namespace="mapper.StudentMapper"/>
    ```
- SELECT语句
  最后在SELECT标签上添加`useCache="true"`属性