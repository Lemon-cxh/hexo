---
title: Docker 部署微服务(Cloud-Admin)以及 Docker 参数、命令
date: 2019-09-05 16:06:55
categories: 
- Docker
tags:
- Docker
- Spring Cloud
description: Docker 的安装、参数、命令以及consul、nacos、redis、rabbitmq、mysql、mogodb的安装和部署微服务
---
1. ## Docker的安装

    1. ### 卸载旧版本
    ```bash
    sudo yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-selinux \
                    docker-engine-selinux \
                    docker-engine
    ```

    2. ### 使用yum安装

    ```bash
    //安装依赖包
    sudo yum install -y yum-utils \
            device-mapper-persistent-data \
            lvm2
    // 添加国内 yum 软件源
    yum-config-manager \
        --add-repo \
        https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
    //更新 yum 软件源缓存，并安装 docker-ce。
    sudo yum makecache fast
    sudo yum install docker-ce
    ```

    3. ### 设置开机自启和自动服务

    ```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    ```

    4. ### 建立docker用户组

    ```bash
    sudo groupadd docker
    //将当前用户加入docker组
    sudo usermod -aG docker $USER
    ```

2. ## 安装consul、nacos、redis、rabbitmq、mysql、mogodb
 
    1. ### 新建docker网络

    ```bash
    docker network create -d bridge spring-net
    ```

    2. ### 获取镜像
    ```bash
    docker pull consul
    docker pull nacos/nacos-server:1.1.4
    docker pull redis
    docker pull rabbitmq:management
    ```

    3. ### 运行镜像

        1. Consul

            ```bash
            docker run -d -p 8500:8500 --name consul --network spring-net consul agent -server -bootstrap-expect=1 -client 0.0.0.0 -ui
            ```
            `-bootstrap-expect`:指定期望的server节点的数量，并当server节点可用的时候，自动进行bootstrapping

        2. Nacos

            ```bash
            docker run --name nacos --network spring-net -e SPRING_DATASOURCE_PLATFORM=mysql -e MYSQL_MASTER_SERVICE_HOST=192.168.1.1 -e MYSQL_MASTER_SERVICE_PORT=3306 -e MYSQL_MASTER_SERVICE_DB_NAME=nacos -e MYSQL_MASTER_SERVICE_USER=user -e MYSQL_MASTER_SERVICE_PASSWORD=password -e MYSQL_DATABASE_NUM=1 -e NACOS_SERVERS="192.168.1.1:8848 192.168.1.2:8848 192.168.1.3:8848" -e NACOS_SERVER_IP=192.168.1.216 -p 8848:8848 -d nacos/nacos-server:1.1.4
            ```

        3. RabbitMQ

            ```bash
            docker run -d --name rabbitmq --network spring-net -e RABBITMQ_DEFAULT_USER=username -e RABBITMQ_DEFAULT_PASS=password -p 15672:15672 -p 5672:5672  rabbitmq:management
            ```

        4. Redis
 
            需要先配置好/etc/redis.conf,可下载[官方配置](http://download.redis.io/redis-stable/redis.conf)。

            密码配置:`requirepass password`

            远程访问注释`bind 127.0.0.1`即可
            ```bash
            docker run -p 6379:6379 --network spring-net -v /etc/redis/redis.conf:/etc/redis/redis.conf -v /opt/docker/redis:/data  --name redis -d redis redis-server /etc/redis/redis.conf --appendonly yes
            ```
            `redis-server --appendonly yes`: 在容器执行redis-server启动命令，并打开redis持久化配置

        5. MySQL

            先配置好/etc/mysql/my.cnf
            ```
            [mysqld]
            # 添加GROUP BY查询限制
            sql_mode = 'STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
            # 拼接最大长度
            group_concat_max_len = 102400
            # 字符集
            character-set-server=utf8mb4
            collation-server=utf8mb4_unicode_ci

            [mysql]
            default-character-set = utf8mb4

            [client]
            default-character-set = utf8mb4
            ```

            注意服务器时区：
            修改文件
            ```bash
            cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
            ```
            在centos7中设置时区的命令可以通过 timedatectl 命令来实现
            ```bash
            timedatectl set-timezone Asia/Shanghai
            ```

            ```bash
            docker run -d -p 3306:3306 --name mysql --network spring-net -v /etc/mysql/my.cnf:/etc/my.cnf -v /var/log/mysql:/var/log/mysql -v /var/lib/mysql:/var/lib/mysql -v /etc/localtime:/etc/localtime:ro -e MYSQL_ROOT_PASSWORD=password mysql:5.7.26
            ```
            进入MySQL容器
            ```bash
            docker exec -it mysql bash
            ```
            登录MySQL
            ```bash
            mysql -u root -p
            ```
            创建远程连接账号
            ```sql
            CREATE USER 'username'@'%' IDENTIFIED BY 'password';
            ```
            授予权限：privileges为ALL，则授予所有权限，可参考[官方文档](https://dev.mysql.com/doc/refman/5.7/en/privileges-provided.html)
            ```sql
            GRANT privileges  ON databasename.* TO 'username'@'%';
            ```
            刷新授权
            ```sql
            flush privileges;
            ```

        6. MogoDB

            创建并启动容器
            ```bash
            docker run -d --name mongo -p 27017:27017 --network spring-net -v /etc/localtime:/etc/localtime:ro -v /var/lib/mongo:/data/db mongo --auth
            ```
            进入容器
            ```bash
            docker exec -it mongo mongo admin
            ```
            创建管理员用户
            ```bash
            db.createUser({ user:'用户名',pwd:'密码',roles:[{ role:'userAdminAnyDatabase', db: 'admin'}]});
            ```
            创建读写用户
            ```bash
            db.createUser({ user: "用户名", pwd: "密码", roles: [{ role: "readWrite", db: "admin" }]})
            ```

3. ## 安装nginx

    可以参考 {% post_link docker-nginx %}

4. ## 部署微服务

    1. ### 配置服务器docker可以远程访问

    ```bash
    //在/usr/lib/systemd/system/docker.service中添加参数
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock

    //重新读取配置文件、重启服务
    systemctl daemon-reload
    systemctl restart docker
    ```

    1. ### pom配置
    ```xml
    <build>
        <finalName>ace-gateway</finalName>
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>1.2.0</version>
                <configuration>
                    <dockerHost>http://你的服务器地址:2375</dockerHost>
                    <imageName>${project.artifactId}</imageName>
                    <dockerDirectory>src/main/docker</dockerDirectory>
                    <resources>
                        <resource>
                            <targetPath>/</targetPath>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>
    ```

    1. ### application配置
    ```yml
    redis:
        host: ${REDIS_HOST:localhost}
    rabbitmq:
        host: ${RABBIT_MQ_HOST:localhost}
    consul:
        host: ${CONSUL_HOST:localhost}
    ```

    1. ### Dockerfile配置
    ```bash
    FROM anapsix/alpine-java:8_server-jre_unlimited
    VOLUME /tmp
    ADD ace-gateway.jar app.jar
    ENV REDIS_HOST=redis
    ENV RABBIT_MQ_HOST=rabbitmq
    ENV CONSUL_HOST=consul
    RUN bash -c 'touch /app.jar' \
        && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
        && echo 'Asia/Shanghai' >/etc/timezone
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-Dfile.encoding=utf-8","-jar","/app.jar"]
    ```

    1. ### 运行服务
    ```bash
    docker run -d --name geteway -p 8765:8765 --network spring-net -v /var/log/gateway:/var/log/gateway iot-gateway
    ```

5. ## 防火墙开启端口
    ```bash
    //查看已开放端口
    firewall-cmd --list-ports
    //添加端口 consul:8500 rabbitmq后台管理界面:15672
    firewall-cmd --zone=public --add-port=8500/tcp --permanent
    //修改配置文件后 使用命令重新加载
    firewall-cmd --reload
    //重启防火墙
    systemctl restart firewalld
    ```

6. ## Docker参数/命令

|参数|描述|
|--|--|
|-d, --detach=false|指定容器运行于前台还是后台，默认为false|
|-i, --interactive=false|打开STDIN，用于控制台交互|
|-t, --tty=false|分配tty设备，该可以支持终端登录，默认为false|
|-u, --user=""|指定容器的用户|
|-a, --attach=[]|登录容器（必须是以docker run -d启动的容器）|
|-w, --workdir=""|指定容器的工作目录|
|-c, --cpu-shares=0|设置容器CPU权重，在CPU共享场景使用|
|-e, --env=[]|指定环境变量，容器中可以使用该环境变量|
|-m, --memory="300M"|设置容器内存上限300M(只设置-m不设置--memory-swap，则--memory-swap为-m两倍)|
|--memory-swap=1|要和-m连用，设置内存+swap的使用限额。，值为-1表示不受限|
|-P, --publish-all=false|指定容器暴露的端口|
|-p, --publish=[]|指定容器暴露的端口|
|-h, --hostname=""|指定容器的主机名|
|-v, --volume=[]|给容器挂载存储卷，挂载到容器的某个目录，要挂载文件需要现在宿主机中提前创建|
|--volumes-from=[]|给容器挂载其他容器上的卷，挂载到容器的某个目录|
|--cap-add=[]|添加权限,--cap-add=SYS_PTRACE(JVM的调试)|
|--cap-drop=[]|删除权限|
|--cidfile=""|运行容器后，在指定文件中写入容器PID值，一种典型的监控系统用法|
|--cpuset=""|设置容器可以使用哪些CPU，此参数可以用来容器独占CPU|
|--device=[]|添加主机设备给容器，相当于设备直通|
|--dns=[]|指定容器的dns服务器|
|--dns-search=[]|指定容器的dns搜索域名，写入到容器的/etc/resolv.conf文件|
|--entrypoint=""|覆盖image的入口点|
|--env-file=[]|指定环境变量文件，文件格式为每行一个环境变量|
|--expose=[]|指定容器暴露的端口，即修改镜像的暴露端口|
|--link=[]|指定容器间的关联，使用其他容器的IP、env等信息|
|--lxc-conf=[]|指定容器的配置文件，只有在指定–exec-driver=lxc时使用|
|--name=""|指定容器名字，后续可以通过名字进行容器管理，links特性需要使用名字|
|--net="bridge"|容器网络设置<br>bridge 使用docker daemon指定的网桥<br> host //容器使用主机的网络<br> container:NAME_or_ID >//使用其他容器的网路，共享IP和PORT等网络资源<br> none 容器使用自己的网络（类似--net=bridge），但是不进行配置|  
|--privileged=false|指定容器是否为特权容器，特权容器拥有所有的capabilities|
|--restart="no"|指定容器停止后的重启策略:<br>no：容器退出时不重启<br>on-failure：容器故障退出（返回值非零）时重启<br>always：容器退出时总是重启|
|--rm=false|指定容器停止后自动删除容器(不支持以docker run -d启动的容器)|
|--sig-proxy=true|设置由代理接受并处理信号，但是SIGCHLD、SIGSTOP和SIGKILL不能被代理|
|--log-opt max-size=10m --log-opt max-file=1|限制生成的json.log单个文件大小和保留文件个数<br>或者配置 Dockerd 的配置：编辑/etc/docker/daemon.json<br>添加{ "log-opts": { "max-size": "10m", "max-file":"5" } }<br>加载配置文件:systemctl daemon-reload重启:systemctl restart docker|

| 命令 |描述  |
|--|--|
|docker attach|附加到正在运行的容器|
|docker commit|从容器的更改创建一个新的映像|
|docker cp|在容器和本地文件系统之间复制文件/文件夹|
|docker create|创建一个新的容器|
|docker diff|检查容器文件系统上文件或目录的更改|
|docker exec|在运行容器中运行命令|
|docker export|将容器的文件系统导出为tar存档|
|docker inspect|显示一个或多个容器的详细信息 docker inspect 容器名 -f '{{json .State}}' |
|docker kill|杀死一个或多个运行容器|
|docker logs|获取容器的日志|
|docker ls|列出容器|
|docker pause|暂停一个或多个容器内的所有进程|
|docker port|列出端口映射或容器的特定映射|
|docker prune|取出所有停止的容器|
|docker rename|重命名容器|
|docker restart|重新启动一个或多个容器|
|docker rm|删除(移除)一个或多个容器<br>-v ：同时移除数据卷|
|docker run|在新容器中运行命令|
|docker start|启动一个或多个停止的容器|
|docker stats|显示容器的实时流资源使用统计信息|
|docker stop|停止一个或多个运行容器|
|docker top|显示容器的正在运行的进程||docker unpause|取消暂停一个或多个容器内的所有流程|
|docker update|更新一个或多个容器的配置|
|docker wait|阻止一个或多个容器停止，然后打印退出代码|
|docker image prune|清除REPOSITORY和TAG为<none>的镜像|
|docker volume prune|清除无主的数据卷|
|docker container update|更新容器属性|