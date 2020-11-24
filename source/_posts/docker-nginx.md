---
title: Docker 中安装 Nginx 以及使用 Logrotate 进行日志轮替
date: 2019-09-02 14:18:32
keywords: Docker,Nginx,Logrotate
categories: 
- Nginx
tags:
- Docker
- Nginx
description: 在Docker中运行Nginx以及使用 Logrotate 进行日志轮替
---
1. ## 安装nginx
    docker 的安装可以参考{% post_link docker-deploy-springCloud %}
    1. ### 获取镜像
```bash
docker pull nginx
```

    2. ### 运行
```bash
docker run -d --rm -p 80:80 --name nginx nginx
```

    3. ### 拷贝配置(只保留nginx.conf以及conf.d)
```bash
docker cp nginx:/etc/nginx /etc/nginx
```

    4. ### 停止容器
```bash
docker stop nginx
```

    5. ### 修改配置文件conf.d下的defalut.conf
```conf
server {
	listen       80;
	server_name  localhost;
	charset utf-8;
	root /opt/nginx/dist;
	index index.html index.htm;
	location /jwt/ {
		# 转发请求到后端服务网关
		proxy_pass http://容器名:8765/jwt/;
	}
	location /api/ {
		proxy_pass http://容器名:8765/api/;
	}
}
```

    6. ### 重新运行镜像
```bash
docker run --name nginx --network spring-net -p 80:80 -d -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -v /etc/nginx/conf.d:/etc/nginx/conf.d -v /opt/nginx/dist:/opt/nginx/dist -v /etc/localtime:/etc/localtime:ro -v /var/log/nginx:/var/log/nginx nginx
# 上传文件到liunx：
> pscp -r E:\**\dist 用户名@服务器地址:/opt/nginx
```

2. ## 使用 Logrotate 进行日志轮替

    1. ### 在宿主机新建脚本`/var/log/nginx/nginx.sh`
```bash
# 不是中止Nginx的进程，而是传递给它信号重新生成日志
kill -USR1 $(/bin/cat /var/run/nginx.pid)
```

    2. ### 新建配置文件`vi /etc/logrotate.d/nginx`
```json
/var/log/nginx/access.log /var/log/nginx/error.log {
	daily
	rotate 60
	compress
	notifempty
	dateext
	sharedscripts
	postrotate
		# docker 中运行 Nginx，执行第一步的脚本
		docker exec -d nginx /bin/sh /var/log/nginx/nginx.sh
		# 宿主机运行 Nginx
		# /bin/kill -USR1 $(/bin/cat /var/run/nginx.pid)
	endscript
}
```

    3. ### Logrotate 配置文件参数详解
| 参数 | 参数说明 |
|--|--|
| daily | 日志的轮替周期是每天 |
| weekly | 日志的轮替周期是每周 |
| monthly | 日志的轮替周期是每月 |
| rotate 数字 | 保留的日志文件个数，0指没有备份 |
| compress | 当进行日志轮替时，对旧的日志进行压缩 |
| create mode owner group | 建立新日志，同时指定新日志的权限与所有者和所属组，如 create 0644 root root |
| mail address| 当进行日志轮替时，输出内容通过邮件发送到指定的邮件地址，如 mail test@gmail.com |
| missingok | 如果日志不存在，则忽略改日志的警告信息 |
| notifempty | 如果日志为空文件，则不进行日志轮替 |
| minsize 大小 | 日志只有大于指定大小才进行日志轮替，而不是按照时间轮替，如 size 100k |
| dateext | 使用日期作为日志轮替文件的后缀，如 access.log-2019-07-12 |
| sharedscripts | 在此关键词之后的脚本只执行一次 |
| prerotate/endscript | 在日志轮替之前执行脚本命令。endscript 标识 prerotate 脚本结束 |
| postrotate/endscript | 在日志轮替之前执行脚本命令。endscript 标识 postrotate脚本结束 |

    4. ### 测试是否有效`logrotate -vf /etc/logrotate.d/nginx`
| 参数 | 参数说明 |
|--|--|
| -?或--help | 在线帮助 |
| -d或--debug | 详细显示指令执行过程，便于排错或了解程序执行的情况 |
| -f或--force | 强行启动记录文件维护操作，纵使logrotate指令认为没有需要亦然 |
| -s<状态文件>或--state=<状态文件> | 使用指定的状态文件 |
| -v或--version | 显示指令执行过程 |
| -usage | 显示指令基本用法 |