---
title: Nginx以及Linux配置
date: 2020-04-14 14:07:39
categories: 
- Nginx
tags:
- Nginx
description: Nginx以及Linux配置优化，提升服务器QPS
---
1. #### 修改用户进程的文件句柄数

    查看当前配置`ulimit -n`，默认为`1024`。修改`/etc/security/limits.conf`文件，添加配置
     ```
    *	hard	nproc	65535
    *	soft	nproc	65535
    *	hard	nofile	65535
    *	soft	nofile	65535
    ```
    完成后再次执行`ulimit -n`查看是否生效了，如果需要设置当前用户session立即生效，执行` ulimit -n 65535`

2. #### Linux TCP连接配置

    编辑`/etc/sysctl.conf`文件

    ```
    net.core.somaxconn = 65535
    net.ipv4.tcp_syncookies = 1
    net.ipv4.tcp_max_syn_backlog = 65535
    net.ipv4.tcp_fin_timeout = 30
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_keepalive_time = 300
    net.ipv4.tcp_max_tw_buckets = 5000
    net.ipv4.ip_local_port_range = 4096 65535
    ```
    执行`sysctl -p`使修改生效

    参数说明：

    - `net.core.somaxconn`——tcp接受队列的长度
    - `net.ipv4.tcp_syncookies`——表示是否打开TCP同步标签,可以防止一个套接字在有过多试图连接到达时引起过载
    - `net.ipv4.tcp_max_syn_backlog`——对于还未获得对方确认的连接请求，可保存在队列中的最大数目
    - `net.ipv4.tcp_fin_timeout`——对于本端断开的socket连接，TCP保持在FIN-WAIT-2状态的时间
    - `net.ipv4.tcp_tw_reuse`——表示是否允许将处于TIME-WAIT状态的socket用于新的TCP连接
    - `net.ipv4.tcp_keepalive_time`——TCP发送keepalive探测消息的间隔时间，用于确认TCP连接是否有效
    - `net.ipv4.tcp_max_tw_buckets`——该参数设置系统的TIME_WAIT的数量，如果超过默认值则会被立即清除
    - `net.ipv4.ip_local_port_range`——规定了tcp/udp可用的本地端口的范围

3. #### Nginx配置

    修改`nginx.conf`文件，可参考[文档](https://nginx.org/en/docs/)

    ```
    worker_processes 8;
    worker_rlimit_nofile 65535;

    error_log /var/log/nginx/error.log warn;
    pid /var/run/nginx.pid;

    events {
        use epoll;
        multi_accept on;
        worker_connections 65535;
    }

    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

        access_log /var/log/nginx/access.log  main;

        charset utf-8;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;

        server_tokens off;
        log_not_found off;
        types_hash_max_size 2048;
        client_max_body_size 16M;

        keepalive_timeout 60;

        open_file_cache max=1024 inactive=60s;

        gzip on;
        gzip_min_length 1k;
        gzip_buffers 4 16k;
        gzip_comp_level 2;
        gzip_http_version 1.0;
        gzip_vary on;
        gzip_proxied any;
        gzip_types text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;

        include /etc/nginx/conf.d/*.conf;
    }
    ```

    | 参数 | 参数说明 |
    |--|--|
    | server_tokens | 隐藏Nginx版本号 |