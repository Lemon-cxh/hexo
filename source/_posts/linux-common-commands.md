---
title: Linux常用命令
date: 2020-06-03 15:54:31
categories: 
- Linux
tags:
- Linux
description: Linux的一些常用命令
---

1. #### firewall

    - 查看
        ```bash
        firewall-cmd --list-all
        ```

    - 开放端口
        ```bash
        firewall-cmd --zone=public --add-port=3000/tcp --permanent
        ```

    - 指定IP开放端口
        ```bash
        firewall-cmd --permanent --remove-rich-rule="rule family="ipv4" source address="111.111.111.111" port protocol="tcp" port="3000" accept"
        ```

    - 重载配置
        ```bash
        firewall-cmd --reload
        ```

2. #### system

    - 设置时区
        ```bash
        timedatectl set-timezone Asia/Shanghai
        ```

    - 显示指定目录所占用空间大小
        ```bash
        du -ah --max-depth=1 /var
        ```

3. #### 运维

    - 根据PID查询具体的应用程序
        ```bash
        ps -ax|grep 20712
        ```