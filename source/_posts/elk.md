---
title: ELK日志收集系统
tags:
  - Elasticsearch
  - Spring Boot
categories:
  - Elasticsearch
description: SpringBoot集成ELK和RabbitMQ完成日志收集
date: 2021-01-20 17:25:03
---


#### 安装Elasticsearch

1. ##### 修改Linux内核配置

    ```bash
    vi /etc/sysctl.conf
    ```

    添加参数

    ```bash
    vm.max_map_count = 262144
    ```

    使配置生效

    ```bash
    sysctl -p
    ```

2.  ##### 获取Elasticsearch配置文件

    拉取镜像

    ```bash
    docker pull elasticsearch:7.10.1
    ```

    启动服务

    ```bash
    docker run -d -it --name elasticsearch --rm -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.10.1
    ```

    进入容器

    ```bash
    docker exec -it elasticsearch bash
    ```

    拷贝配置文件并停止容器

    ```bash
    docker cp elasticsearch:/usr/share/elasticsearch/config /usr/share/elasticsearch;
    docker stop elasticsearch
    ```

    创建文件夹用于绑定数据卷

    ```bash
    mkdir /usr/share/elasticsearch/data
    chmod g+rwx /usr/share/elasticsearch/data
    chmod -R 777 /usr/share/elasticsearch/config/certs
    ```

3. ##### 启动容器修改预置账号的密码

    ```bash
    docker run -d -it --name elasticsearch --net somenetwork -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -v /usr/share/elasticsearch/data:/usr/share/elasticsearch/data -v /usr/share/elasticsearch/config:/usr/share/elasticsearch/config elasticsearch:7.10.1
    ```

    进入容器

    ```bash
    docker exec -it elasticsearch bash
    ```

    修改`/usr/share/elasticsearch/config/elasticsearch.yml`

    ```yml
    # 添加安全配置
    xpack.security.enabled: true
    ```

    修改预置账号的密码

    ```bash
    ./bin/elasticsearch-setup-passwords interactive
    ```

    - `apm_system`APM系统专用账号
    - `kibana_system`仅可用于kibana用来连接elasticsearch并与之通信, 不能用于kibana登录
    - `kibana`Kibana访问专用账号
    - `logstash_system`Logstash访问专用账号
    - `beats_system`FileBeat访问专用账号
    - `remote_monitoring_user`远程监控账号
    - `elastic`超级管理员账号

    如果使用密码保护节点的证书，请将密码添加到密钥存储
    ```bash
    .bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
    .bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
    ```

4. ##### 配置Elasticsearch

    生成节点证书

    ```bash
    # 创建目录存放证书
    mkdir config/certs
    # 创建密钥
    ./bin/elasticsearch-certutil ca
    # 输入所需的输出文件目录
    config/certs/elastic-stack-ca.p12
    # 生成证书
    ./bin/elasticsearch-certutil cert --ca config/certs/elastic-stack-ca.p12
    # 输入所需的输出文件目录
    config/certs/elastic-certificates.p12
    # 从文件中提取 CA 证书链
    openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elasticsearch-ca.pem
    ```

     修改`vi `/usr/share/elasticsearch/config/jvm.options`

    ```
    # 内存分配不超过机器的一半
    # 确保堆内存最小值(Xms)与最大值(Xmx)的大小是相同的，防止程序在运行时改变堆内存大小， 这是一个很耗系统资源的过程。
    -Xms1g
    -Xmx1g
    ```

    修改`/usr/share/elasticsearch/config/elasticsearch.yml`

    ```yml
    cluster.name: "elasticsearch-test"
    network.host: 0.0.0.0
    bootstrap.memory_lock: true
    xpack.security.enabled: true
    # 加密 HTTP 客户端通信
    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
    xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12
    # 加密集群中节点之间的通讯以及Elasticsearch和Kibana之间通信
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12
    ```

#### 安装Kibana

1. ##### 配置Kibana

    拉取镜像
    ```bash
    docker pull kibana:7.10.1
    ```

    启动容器

    ```bash
    docker run -d -it --name kibana --rm -p 5601:5601 kibana:7.10.1
    ```

    拷贝配置文件并停止容器

    ```bash
    docker cp kibana:/usr/share/kibana/config /usr/share/kibana;
    docker stop kibana
    ```

    修改`/usr/share/kibana/config/kibana.yml`

    ```yml
    server.name: kibana
    server.host: "0"
    server.ssl.enabled: true
    server.ssl.keystore.path: "/usr/share/elasticsearch/config/certs/elastic-certificates.p12"
    server.ssl.keystore.password: ""
    elasticsearch.hosts: [ "https://IP:9200" ]
    elasticsearch.username: "kibana_system"
    elasticsearch.password: "安装elasticsearch时设置的kibana_system密码"
    elasticsearch.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    elasticsearch.ssl.truststore.password: ""
    elasticsearch.ssl.verificationMode: certificate
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/elasticsearch/config/certs/elasticsearch-ca.pem"]
    monitoring.ui.container.elasticsearch.enabled: true
    xpack.encryptedSavedObjects.encryptionKey: 32位字符串
    xpack.reporting.capture.browser.chromium.disableSandbox: false
    # 设置为中文
    i18n.locale: "zh-CN"
    ```

1. ##### 启动容器

    ```bash
    docker run -d -it --name kibana --net somenetwork -p 5601:5601 -v /usr/share/kibana/config:/usr/share/kibana/config -v /usr/share/elasticsearch/config/certs:/usr/share/elasticsearch/config/certs kibana:7.10.1
    ```

2. ##### 后台管理配置Kibana

   1. ###### 为Logstash设置身份验证凭据

       在Kibana中的`管理`>`安全`>`角色`中创建`logstash_writer`角色

       [logstash_writer](logstash_writer.png)

       在Kibana中的`管理`>`安全`>`用户`中创建`logstash_internal`用户，设置角色为`logstash_writer`。

    1. ###### 配置时间格式

        在Kibana中的`管理`>`高级设置`>`常规`>`Date format`和`Date with nanoseconds format`中配置时间格式为`yyyy-MM-DD HH:mm:ss.SSS`


#### 安装Logstash

1. ##### 配置Logstash

    拉取镜像

    ```bash
    docker pull logstash:7.10.1
    ```

    启动容器

    ```bash
    #  通过在名为LOGSTASH_KEYSTORE_PASS的环境变量中存储密码来保护对 Logstash 密钥存储的访问
    docker run -d -it --name logstash --rm -p 5047:5047 -e LOGSTASH_KEYSTORE_PASS=密码 logstash:7.10.1
    ```

    创建密钥库存储密钥

    ```bash
    # 进入容器
    docker exec -it logstash bash
    # 创建密钥库
    ./bin/logstash-keystore create
    # 添加键
    ./bin/logstash-keystore add ES_PWD
    # 输入密码为logstash_internal用户的密码
    ```

    拷贝配置文件并停止容器

    ```bash
    docker cp logstash:/usr/share/logstash/config /usr/share/logstash;
    docker stop logstash;
    mkdir /usr/share/logstash/data;
    chmod 777 /usr/share/logstash/data
    ```

    修改`/usr/share/logstash/config/logstash.yml`

    ```yml
    http.host: "0.0.0.0"
    xpack.monitoring.elasticsearch.hosts: [ "http://ip:9200" ]
    xpack.monitoring.elasticsearch.username: logstash_system
    xpack.monitoring.elasticsearch.password: logstash_system的密码
    ```

    修改`/usr/share/logstash/pipeline/logstash.conf`

    ```conf
    input {
        rabbitmq {
            durable => true
            exchange => "logs"
            exchange_type => "topic"
            key => "logstash"
            host => "IP"
            port => 5672
            user => "用户名"
            password => "密码"
            # 自定义虚拟网络
            vhost => logs
        }
    }


    filter {
        if [message] {
            kv { }
        }
    }


    output {
        elasticsearch {
            hosts => ["https://IP:9200"]
            user => "logstash_internal"
            password => "${ES_PWD}"
            ssl => true
            ssl_certificate_verification => false
            truststore => '/usr/share/elasticsearch/config/certs/elastic-certificates.p12'
            truststore_password => ""
        }
    }
    ```

    > 自定义虚拟网络logs
    > 创建logs：rabbitmqctl add_vhosts logs
    > 赋予用户权限：rabbitmqctl set_permissions -p logs 用户名 ".*" ".*" ".*"
    > input.rabbitmq的配置参考[官方文档](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-rabbitmq.html#plugins-inputs-rabbitmq-arguments)


2. ##### 启动容器

    ```bash
    docker run -d -it --name logstash --net somenetwork -p 5044:5044 -e LOGSTASH_KEYSTORE_PASS=密码 -v /usr/share/logstash/config:/usr/share/logstash/config -v /usr/share/logstash/pipeline:/usr/share/logstash/pipeline -v /usr/share/logstash/data:/usr/share/logstash/data logstash:7.10.1
    ```

#### 配置Spring Boot

1. ##### 配置logback-spring.xml

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration>

        <springProperty scope="context" name="logstash.address" source="logstash.address"/>
        <springProperty scope="context" name="spring.cloud.nacos.discovery.ip" source="spring.cloud.nacos.discovery.ip"/>
        <springProperty scope="context" name="spring.application.name" source="spring.application.name"/>
        <springProperty scope="context" name="logging.level.root" source="logging.level.root"/>
        <springProperty scope="context" name="logging.path" source="logging.path"/>
        <springProperty scope="context" name="mq.host" source="spring.rabbitmq.host"/>
        <springProperty scope="context" name="mq.port" source="spring.rabbitmq.port"/>
        <springProperty scope="context" name="mq.username" source="spring.rabbitmq.username"/>
        <springProperty scope="context" name="mq.password" source="spring.rabbitmq.password"/>

        <appender name="LOGSTASH" class="org.springframework.amqp.rabbit.logback.AmqpAppender">
            <layout>
                <pattern><![CDATA[{"service":"${spring.application.name}",
                    "serviceIp":"${spring.cloud.nacos.discovery.ip}",
                    "dateTime":"%d{yyyy-MM-dd HH:mm:ss.SSS}",
                    "level":"%-5level",
                    "thread":"%thread",
                    "requestId":"%X{requestId}",
                    "logger":"%logger{50}",
                    "message":"%replace(%msg){'\"','\\\"'}",
                    "stack_trace":"%replace(%exception){'[\r\t\n]',''}"}]]></pattern>
            </layout>
            <host>${mq.host}</host>
            <port>${mq.port}</port>
            <username>${mq.username}</username>
            <password>${mq.password}</password>
            <applicationId>${spring.application.name}</applicationId>
            <declareExchange>true</declareExchange>
            <routingKeyPattern>logstash</routingKeyPattern>
            <charset>UTF-8</charset>
            <contentType>application/json</contentType>
            <virtualHost>logs</virtualHost>
        </appender>

        <appender name="ASYNC_LOGSTASH" class="ch.qos.logback.classic.AsyncAppender">
            <discardingThreshold>0</discardingThreshold>
            <queueSize>256</queueSize>
            <appender-ref ref="LOGSTASH"/>
        </appender>

        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>${logging.path}</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <FileNamePattern>${logging.path}.%d{yyyy-MM-dd}.%i.gz</FileNamePattern>
                <MaxHistory>180</MaxHistory>
                <maxFileSize>10MB</maxFileSize>
            </rollingPolicy>
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
                <charset>UTF-8</charset>
            </encoder>
        </appender>

        <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
            <discardingThreshold>0</discardingThreshold>
            <queueSize>256</queueSize>
            <appender-ref ref="FILE"/>
        </appender>

        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            </encoder>
        </appender>


        <root level="${logging.level.root}">
            <appender-ref ref="ASYNC_LOGSTASH"/>
            <appender-ref ref="ASYNC_FILE"/>
            <appender-ref ref="CONSOLE"/>
        </root>
    </configuration>
    ```

2. ##### 日志切面

```java

    @Before("***")
    public void apiLogDoBefore(JoinPoint joinPoint) {
        // 设置 请求ID
        MDC.put(RequestConstants.REQUEST_ID, httpServletRequest.getHeader(RequestConstants.X_REQUEST_ID));
        logger.info("ip={} class={} method={} params={}",
                ClientUtil.getClientIp(httpServletRequest),
                joinPoint.getTarget().getClass().getName(),
                joinPoint.getSignature().getName(),
                JSON.toJSONString(joinPoint.getArgs(), SerializerFeature.UseSingleQuotes));
    }

    @AfterReturning(returning = "ret", value = "***")
    public void doAfterReturning(Object ret) {
        logger.info(JSON.toJSONString(ret, SerializerFeature.UseSingleQuotes));
        MDCAdapter mdcAdapter =  MDC.getMDCAdapter();
        if (Objects.nonNull(mdcAdapter)) {
            mdcAdapter.clear();
        }
    }

}
```