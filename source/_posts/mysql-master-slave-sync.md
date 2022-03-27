---
title: MySQL主从同步
date: 2019-12-09 17:53:33
keywords: MySQL,主从同步
categories: 
- MySQL
tags:
- MySQL
description: MySQL配置主从同步
---

#### MySQL的复制

 主服务器将更新写入二进制文件，并维护文件的一个索引以跟踪日志循环。这些日志可以记录发送到从服务器的更新。当一个从服务器连接主服务器时，它通知主服务器从服务器在日志中读取的最后一次成功更新的位置。从服务器接收从那时起发生的任何更新，然后封锁并等待主服务器通知新的更新。

 MySQL支持的复制类型：

  1. 基于语句的复制(逻辑复制)：在主服务器上执行的SQL语句，在从服务器上执行同样的语句。MySQL默认采用基于语句的复制，效率比较高。一旦发现没法精确复制时，会自动选择基于行的复制。
  2. 基于行的复制：把改变的内容复制过去，而不是把命令再从服务器上执行一遍。
  3. 混合类型的复制：默认采用基于语句的复制，一旦发现基于语句无法精确复制时，会自动选择基于行的复制。

复制主要有三个步骤：

  1. 在主库上把数据更改记录到二进制日志(Binary Log)中
  2. 从库将主库上的日志复制到自己的中继日志(Relay Log)中
  3. 从库读取中继日志中的事件，并将其重放到从库中

#### 具体操作

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
    # 保留指定日期范围内的bin log历史日志。默认为0，保留所有日志
    expire_logs_days=7
    # bin log日志每达到设定大小后，会使用新的bin log日志。默认为1073741824，1GB
    max_binlog_size=1073741824
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

#### show slave status 参数


| 参数 | 值 | 说明 |
| :-: | :-: | :-: |
|Slave_IO_State |  Waiting | for master to send event
|                  Master_Host |  192.168.1.80 |
|                  Master_User |  repl |
|                  Master_Port |  3306 |
|                Connect_Retry |  60 |
|              Master_Log_File |  mysql-bin.000083 |
|          Read_Master_Log_Pos |  393099157 |
|               Relay_Log_File |  ecs-bb76-0006-relay-bin.000235 |
|                Relay_Log_Pos |  300045860 |
|        Relay_Master_Log_File |  mysql-bin.000083 |
|             Slave_IO_Running |  Yes |
|            Slave_SQL_Running |  Yes |
|              Replicate_Do_DB |   |
|          Replicate_Ignore_DB |   |
|           Replicate_Do_Table |   |
|       Replicate_Ignore_Table |   |
|      Replicate_Wild_Do_Table |   |
|  Replicate_Wild_Ignore_Table |   |
|                   Last_Errno |  0 |
|                   Last_Error |   |
|                 Skip_Counter |  0 |
|          Exec_Master_Log_Pos |  393099157 |
|              Relay_Log_Space |  300322434 |
|              Until_Condition |  None |
|               Until_Log_File |   |
|                Until_Log_Pos |  0 |
|           Master_SSL_Allowed |  No |
|           Master_SSL_CA_File |   |
|           Master_SSL_CA_Path |   |
|              Master_SSL_Cert |   |
|            Master_SSL_Cipher |   |
|               Master_SSL_Key |   |
|        Seconds_Behind_Master |  0 |
|Master_SSL_Verify_Server_Cert |  No |
|                Last_IO_Errno |  0 |
|                Last_IO_Error |   |
|               Last_SQL_Errno |  0 |
|               Last_SQL_Error |   |
|  Replicate_Ignore_Server_Ids |   |
|             Master_Server_Id |  80 |
|                  Master_UUID |  d2c28dea-8939-11e9-a1af-fa163eea46ba |
|             Master_Info_File |  /var/lib/mysql/master.info |
|                    SQL_Delay |  0 |
|          SQL_Remaining_Delay |  NULL |
|      Slave_SQL_Running_State |  Slave has read all relay log; waiting for more updates |
|           Master_Retry_Count |  86400 |
|                  Master_Bind |   |
|      Last_IO_Error_Timestamp |   |
|     Last_SQL_Error_Timestamp |   |
|               Master_SSL_Crl |   |
|           Master_SSL_Crlpath |   |
|           Retrieved_Gtid_Set |   |
|            Executed_Gtid_Set |   |
|                Auto_Position | 0 |
|         Replicate_Rewrite_DB |   |
|                 Channel_Name |   |
|           Master_TLS_Version |  |
#### 常用命令

- `show relaylog events in "Relay_Log_File" from Relay_Log_Pos limit n;`：查看relaylog events
- `set GLOBAL SQL_SLAVE_SKIP_COUNTER = 1;`：跳过为一个Binlog event group，也就是跳过一个事务