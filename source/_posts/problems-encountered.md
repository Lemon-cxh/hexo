---
title: 遇到的问题
date: 2020-04-10 14:14:19
tags:
- problems
description: 开发过程中遇到的问题
---
1. ##### Spring Cloud Gateway

    1. #### java.lang.IllegalArgumentException: Invalid character '[' for QUERY_PARAM in "match[]"
       Spring Cloud Gateway与Webflux的兼容性的问题，项目的Gateway版本2.0.0.RELEASE，去除`spring-boot-starter-webflux`依赖解决
       可以升级Gateway版本。[issues](https://github.com/spring-cloud/spring-cloud-gateway/issues/462)

    2. #### io.netty.handler.codec.EncoderException: java.lang.IllegalStateException: unexpected message type: DefaultHttpRequest
        Spring Cloud Gateway版本2.0.1.RELEASE更新为2.0.4.RELEASE。[issues](https://github.com/reactor/reactor-netty/issues/177)