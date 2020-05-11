---
title: MySQL主从同步
date: 2019-12-09 17:53:33
categories: 
- MySQL
tags:
- MySQL
description: MySQL配置主从同步
---
1. ##### 配置主库/etc/my.cnf,需要重启MySQL服务

    ```ini
    [mysqld]
    server-id = 1
    # 唯一ID，master/slave 集群中不能重复
    log-bin = mysql-bin
    # master开启二进制日志，默认记录所有库所有表的操作。
    # 不同步哪些数据库
    binlog-ignore-db = mysql  
    binlog-ignore-db = test  
    binlog-ignore-db = information_schema  
    # 只同步哪些数据库，除此之外，其他不同步  
    binlog-do-db = game
    ```

2. ##### 主库创建用于同步的账号

    ```sql
    grant replication slave on *.* to 'username'@'192.168.1.%'IDENTIFIED BY 'password';
    ```
    其中192.168.1.%使用通配符，例如192.168.1.%表192.168.1.0-192.168.1.255这个网段所有slave都可以能通过该户来访问master。

3. ##### 备份主库

    锁定主库为只读状态，防止在备份数据库时外部程序对数据修改
    ```sql
    flush tables with read lock;
    ```
    查看master的状态，同时记录File和Position
    ```sql
    show master status;
    ```
    备份数据库
    ```bash
    mysqldump -uroot -p dbname > db.sql
    ```
    主库恢复写操作
    ```sql
    unlock tables;
    ```
4. ##### 导入数据到从库
    ```bash
    mysql -uroot -p dbname < db.sql
    ```

5. ##### 配置从库/etc/my.cnf
    ```ini
    [mysqld]
    server-id=2  #设置server-id,必须唯一
    ```
    配置后启动MySQL
    ```bash
    service mysql start --skip-slave-start
    ```
6. ##### 从库启动主从同步
    执行同步SQL语句
    ```sql
    CHANGE MASTER TO MASTER_HOST='主库IP',MASTER_USER='username',MASTER_PASSWORD='password',MASTER_LOG_FILE='第三布记录的File',MASTER_LOG_POS=第三步记录的Position;
    ```
    启动slave同步进程
    ```sql
    start slave;
    ```
    查看主从同步状态
    ```sql
    show slave status\G
    ```
    如下两行显示Yes即为正在运行
    ```
    Slave_IO_Running: Yes
    Slave_SQL_Running: Yes
    ```