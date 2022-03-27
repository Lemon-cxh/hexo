---
title: 运维常用命令
date: 2020-06-03 15:54:31
keywords: 运维命令
categories: 
- Linux
tags:
- Linux
description: 运维时Linux，MySQL的一些常用命令
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

    - 查看 CentOS 版本号
        ```bash
        cat /etc/redhat-release
        ```

    - 查看 CPU 总的线程数(逻辑 CPU 数量)
        ```bash
        grep 'processor' /proc/cpuinfo | sort -u | wc -l
        ```

    - 查看网络系统状态信息
        ```bash
        # 查看CLOSE_WAIT的总数
        netstat -antp | grep CLOSE_WAIT|wc -l
        # 查看CLOSE_WAIT的信息
        netstat -antop | grep CLOSE_WAIT
        # 根据TCP状态分组查询
        netstat -na | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
        ```

    - 根据PID查询具体的应用程序
        ```bash
        ps -ax|grep 20712
        ```

    - 查询日志
        ```bash
        # 查看9日这天的所有日志包含 getLog][123456 的信息
        zcat service.log.2020-09-09.*.gz|grep getLog\\]\\[123456
        ```

4. #### MySQL
   
    - 查询时间段内的binlog日志
        ```bash
        mysqlbinlog --no-defaults --base64-output=DECODE-ROWS -v  /var/lib/mysql/mysql-bin.000001 --start-datetime '2020-10-09 15:00:00' --stop-datetime '2020-10-09 16:00:00' > /tmp/mysql.sql
        ```

    - 导出库/表
      ```bash
      mysqldump -uroot -p --databases 库名 --tables 表名  > /var/log/mysql/***.sql
      ```

    - 显示用户正在运行的线程
        ```bash
        show full processlist;
        SELECT * FROM INFORMATION_SCHEMA.processlits;
        ```

    - 查看死锁信息
        ```bash
        show engine innodb status;
        ```

    - 赋予用户权限
        ```bash
        GRANT SELECT, UPDATE ON `库名`.* TO 'user'@'%';
        ```
        刷新权限
        ```bash
        FLUSH PRIVILEGES;
        ```
        常用权限:
        |权限|说明|
        |:-:|:-:|
        | ALL | 授予除了GRANT OPTION之外的指定访问级别的所有权限 |
        | GRANT OPTION | 允许用户有权授予或撤销其他帐户的权限 |
        | CREATE USER | 允许用户使用CREATE USER，DROP USER，RENAME USER和REVOKE ALL PRIVILEGES |
        | PROCESS | 允许用户使用SHOW PROCESSLIST语句查看所有进程 |
        | RELOAD | 允许用户使用FLUSH操作 |
        | ALTER | 允许用户使用ALTER TABLE语句 |
        | INDEX | 允许用户创建或删除索引 |
        | CREATE | 允许用户创建数据库和表 |
        | DROP | 允许用户删除数据库，表和视图 |
        | DELETE | 允许用户使用DELETE |    
        | INSERT | 允许用户使用INSERT语句 |
        | SELECT | 允许用户使用SELECT语句 |
        | UPDATE | 允许用户使用UPDATE语句 |

5. #### Reids

    - 批量删除
        ```bash
        redis-cli -a 密码 keys "KEY:*" | xargs redis-cli -a 密码 del
        ```