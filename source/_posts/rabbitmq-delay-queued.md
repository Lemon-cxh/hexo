---
title: RabbitMQ延迟队列
date: 2020-08-12 17:47:29
keywords: RabbitMQ
categories: 
- Spring Boot
tags:
- Spring Boot
- RabbitMQ
description: RabbitMQ的延迟队列实现
---

- #### 1. RabbitMQ配置

    ```yaml
    rabbitmq:
        host: 127.0.0.1
        port: 5672
        username: admin
        password: admin
        listener:
        simple:
            prefetch: 1
            acknowledge-mode: manual
        direct:
            prefetch: 1
            acknowledge-mode: manual
    ```

- #### 2. 延迟队列

    ```java
    // 声明死信Exchange
    @Bean(RabbitConstant.DEAD_LETTER_EXCHANGE)
    public DirectExchange deadLetterExchange(){
        return (DirectExchange) ExchangeBuilder.directExchange(RabbitConstant.DEAD_LETTER_EXCHANGE).durable(true).build();
    }

    // 声明延时队列 延时5s
    @Bean(RabbitConstant.OPEN_SERVICE_EVENT_DEAD_LETTER_QUEUE)
    public org.springframework.amqp.core.Queue deadLetterQueue(){
        return QueueBuilder.durable(RabbitConstant.OPEN_SERVICE_EVENT_DEAD_LETTER_QUEUE)
                .withArgument("x-dead-letter-exchange", RabbitConstant.OPEN_SERVICE_EXCHANGE)
                .withArgument("x-dead-letter-routing-key", RabbitConstant.OPEN_SERVICE_EVENT_KEY)
                .withArgument("x-message-ttl", 5000L)
                .build();
    }

    // 声明延时队列绑定关系
    @Bean
    public Binding businessBindingA(@Qualifier(RabbitConstant.OPEN_SERVICE_EVENT_DEAD_LETTER_QUEUE)Queue queue,
                                    @Qualifier(RabbitConstant.DEAD_LETTER_EXCHANGE) DirectExchange directExchange){
        return BindingBuilder.bind(queue).to(directExchange).with(RabbitConstant.OPEN_SERVICE_EVENT_KEY);
    }

    // 声明死信队列 用于接收延时处理的消息
    @RabbitListener(bindings = @QueueBinding(value = @Queue(RabbitConstant.OPEN_SERVICE_EVENT_QUEUE),
        exchange = @Exchange(RabbitConstant.OPEN_SERVICE_EXCHANGE),
        key = RabbitConstant.OPEN_SERVICE_EVENT_KEY))
    public void test(Message message, Channel channel) throws IOException {
        MessageDTO<Entity> messageDTO = JSON.parseObject(message.getBody(),
                new TypeReference<MessageDTO<Entity>>() {
                }.getType());
        log.info(JSON.toJSONString(messageDTO));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
    ```

- #### 3. 发送延迟消息

    ```java
    MessageDTO<Entity> messageDTO = MessageDTO.<Entity>builder()
                    .type(RabbitTypeConstant.OPEN_INSERT)
                    .data(data)
                    .build();
    rabbitTemplate.convertAndSend(RabbitConstant.DEAD_LETTER_EXCHANGE,
                    RabbitConstant.OPEN_SERVICE_EVENT_KEY,
                    MessageBuilder.withBody(JSON.toJSONBytes(messageDTO)).build());
    ```