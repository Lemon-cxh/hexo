---
title: rabbitmq
categories: 
- RabbitMQ
tags:
- RabbitMQ
- SpringBoot
description: SpringBoot集成RabbitMQ以及RabbitMQ的一些知识点
---
1. #### SpringBoot集成RabbitMQ
    1. ##### pom.xml添加依赖
        ```xml
        <dependency>
            <groupId>org.springframework.amqp</groupId>
            <artifactId>spring-rabbit</artifactId>
        </dependency>
        ```

    2. ##### application.yml配置
        ```yml
        spring:
            rabbitmq:
                host: ${RABBIT_MQ_HOST:127.0.0.1}
                port: ${RABBIT_MQ_PORT:5672}
                username: admin
                password: admin
                publisher-confirms: true #支持发布确认
                publisher-returns: true  #支持发布返回
                listener:
                simple:
                    acknowledge-mode: manual #采用手动应答
                    concurrency: 1 #指定最小的消费者数量
                    max-concurrency: 1 #指定最大的消费者数量
                    retry:
                    enabled: true #是否支持重试
        ```