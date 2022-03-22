---
title: 遇到的问题
date: 2020-04-10 14:14:19
keywords: problems,问题
tags:
- problems
description: 开发过程中遇到的问题,以及处理方式
---
1. ##### Spring Boot

    1. ###### 服务启动后即关闭，没有错误日志

        ```java
        public static void main(String[] args) {
            try {
                SpringApplication.run(TestBootstrap.class, args);
            } catch (Exception e) {
                log.error("error", e);
            }
        }
        ```

2. ##### Spring Cloud Gateway

    1. ###### Invalid character '[' for QUERY_PARAM in "match[]"

        ```vim
        java.lang.IllegalArgumentException: Invalid character '[' for QUERY_PARAM in "match[]"
        ```
        Spring Cloud Gateway与Webflux的兼容性的问题，项目的Gateway版本2.0.0.RELEASE，去除`spring-boot-starter-webflux`依赖解决可以升级Gateway版本。[issues](https://github.com/spring-cloud/spring-cloud-gateway/issues/462)

    2. ###### unexpected message type: DefaultHttpRequest

        ```vim
        io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: unexpected message type: DefaultHttpRequest
        ```
        Spring Cloud Gateway以及Spring Boot版本2.0.1.RELEASE更新为2.0.4.RELEASE。[issues](https://github.com/reactor/reactor-netty/issues/177)

3. ##### Nginx

    1. ###### upstream sent no valid HTTP/1.0 header while reading response header from upstream

        Nginx报错，然后设置了Nginx和业务服务配置
        ```conf
        proxy_http_version 1.1;
        proxy_cache_bypass      $http_upgrade;

        proxy_set_header Upgrade                $http_upgrade;
        proxy_set_header Connection "";
        proxy_set_header Host                   $host;
        proxy_set_header X-Real-IP              $remote_addr;
        proxy_set_header X-Forwarded-For        $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto      $scheme;
        proxy_set_header X-Forwarded-Host       $host;
        proxy_set_header X-Forwarded-Port       $server_port;
        ```
        ```yml
        server:
            use-forward-headers: true
            tomcat:
                remote-ip-header: x-forwarded-for
                protocol-header: x-forwarded-proto
                port-header: X-Forwarded-Port
        ```
        
        然后依旧报错，此时把服务停止运行再次请求，然而正常访问，说明Spring Gateway和Nginx之间没有问题，便排查业务服务。
        在`HandlerInterceptorAdapter`处理拦截器种发现了这么一段代码`response.sendError`。
        此方法使用指定的状态码发送一个错误响应至客户端。服务器默认会创建一个HTML格式的服务错误页面作为响应结果，其中包含参数msg指定的文本信息，这个HTML页面的内容类型为`text/html`，保留cookies和其他未修改的响应头信息。
        最后去掉此行代码，替换为抛出全局异常，得以解决。
        ```java
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            ***
            response.sendError(***);
            ***
        }
        ```

    2. ###### upstream server temporarily disabled while reading response header from upstream
    3. ###### no live upstreams while connecting to upstream

        ```conf
        upstream zxjl {
            least_conn;
            server 192.168.1.1:8765 max_fails=3 fail_timeout=30s;
            server 192.168.1.2:8765 max_fails=3 fail_timeout=30s;
            keepalive 256;
        }
        ```

        > max_fails 默认值为1, fail_timeout 默认值为 10 秒。

        fail 失败会返回:
        - upstream timed out (110: Connection timed out) while reading response header from upstream
        - connect() failed (111: Connection refused) while connecting to upstream

4. ##### 服务器

    1. ###### Error: Network Error
        请求时不时提示net:ERR_CONNECTION_TIMED_OUT以及Error: Network Error。
        ```conf
        # /etc/sysctl.conf
        net.ipv4.tcp_tw_recycle = 0
        ```

5. ##### MySQL

    1. ###### Slave_SQL_Running: No

        因为设置主从同步时忘记排除mysql库，导致主从同步故障。
        ```bin
        slave stop;
        # 跳过一个复制错误
        set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
        slave start;
        ```