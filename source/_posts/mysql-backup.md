---
title: MySQL 定时备份
date: 2020-10-09 14:49:03
categories: 
- MySQL
tags:
- MySQL
description: 设置 MySQL 数据的定时备份
---

新建脚本

    ```bash
    vi /usr/local/mysql/mysql-backup.sh
    ```

备份脚本，以及删除7日前数据

    ```bash
    db_user="root"
    db_pwd="root"
    db_name="test"
    bak_dir="/usr/local/mysql/backup"
    time="$(date +"%Y%m%d")"
    mysqldump -u$db_user -p$db_pwd -h 127.0.0.1 $db_name > $bak_dir/${db_name}_$time.sql;
    find $bak_dir -name "$db_name*.sql" -type f -mtime +7 -exec rm -rf {} \; > /dev/null 2>&1;
    ```

修改权限

    ```bash
    chmod 777 /usr/local/mysql/backup/mysql-backup.sh
    ```

设置定时任务

    ```bash
    crontab -e
    ```

每天两点执行

    ```bash
    0 2 * * * /usr/local/mysql/backup/mysql-backup.sh
    ```

重启服务

    ```bash
    service crond restart
    ```

