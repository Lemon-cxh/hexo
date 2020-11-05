---
title: Jenkins + Gogs 自动化部署
date: 2019-09-29 16:53:54
categories: 
- Jenkins
tags:
- Jenkins
description: 使用Docker运行Jenkins,结合Gogs完成Java和Vue项目的自动化部署,最后使用企业微信通知部署进度
---
> 使用Docker运行Jenkins,结合Gogs完成Java和Vue项目的自动化部署
> Docker的安装可以参考{% post_link docker-deploy-springCloud %}

1. #### 安装以及配置Jenkins

    1. ##### 安装Jenkins
        新建目录
        ```bash
        mkdir /var/jenkins_home
        ```
        修改所属用户
        ```bash
        chown 1000 /var/jenkins_home
        ```
        没有此ID用户则添加
        ```bash
        useradd -u 1000 jenkins
        ```
        运行容器
        ```bash
        docker run -d -i --name jenkins --network docker-net -p 8080:8080 -p 50000:50000 -v /var/jenkins_home:/var/jenkins_home -v /etc/localtime:/etc/localtime:ro jenkins/jenkins:lts
        # 配置Nginx反向代理可以添加参数
        --env JENKINS_OPTS="--prefix=/jenkins"
        # Nginx 配置
        location /jenkins {
            proxy_pass http://172.17.0.1:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_redirect  off;
        }
        ```
        查看日志 会实时显示日志，最后会显示其中类似f3f67320f92e4ba6bdecb083a03af92c的一行即为密码
        ```bash
        docker logs -f jenkins
        ```

    2. #####  配置Jenkins

        访问<http://localhost:8080>(Nginx反向代理访问<http://localhost/jenkins>)输入上一步的密码，选择第一种安装插件方式即可

        如果提示`该jenkins实例似乎已离线`:

        - 访问<http://localhost:8080/pluginManager/advanced>找到最底部的Update Site选项，修改URL的https为http。

        - 修改`/var/jenkins_home/updates/default.json`文件的`connectionCheckUrl`属性的值，改为可访问地址,重启Jenkins

        1. ###### 安装插件
        系统管理 -> 插件管理 -> 可选插件
        搜索安装`Generic Webhook Trigger`, `NodeJS Plugin`, `Maven Integration`, `Publish Over SSH`
        
        2. ###### 配置
        - 系统管理 -> 系统设置 -> 新增Publish over SSH配置
        ![SSH Servers配置](ssh-config.png)

        - 系统管理 -> 全局工具配置 -> Maven安装以及NodeJS安装,勾选自动安装即可
        ![SSH Servers配置](tool-config.png)

2. #### 自动部署Gogs上的Java项目
    > 目前项目为微服务，使用docker部署，并且只自动部署测试分支，所以以下例子只有推送release分支且指定模块下有修改才触发自动部署。

    1. ##### 新建任务
        新建任务 -> 输入名称 -> 构建一个maven项目 -> 确定 -> 输入描述。

    2. ##### 输入源码管理信息
        ![源码管理](build-gogs.png)
    
    3. ##### 构建触发器
        - 勾选`Generic Webhook Trigger`
        ![Generic Webhook Trigger](build-trigger1.png)
        - 新增三个Post content parameters
        ![Post content parameters](build-trigger2.png)
        ![Post content parameters](build-trigger3.png)
        ![Post content parameters](build-trigger4.png)
        - 输入Token
        ![输入Token](build-trigger5.png)

    4. ##### Optional filter
        ![Optional filter](build-filter.png)

    5. ##### Pre Steps以及Build
        ![maven](build-maven.png)

    6. ##### Post Steps
        ![Post Steps](build-post-steps.png)

    7. ##### 登录Gogs在仓库管理 -> 管理Web钩子 -> 添加Web钩子 -> gogs

        推送地址为：http://localhost:8080/generic-webhook-trigger/invoke?token=`构建触发器中输入的Token`
        ![gogs config](gogs-config.png)

3. #### 自动部署Gogs上的Vue项目
    1. ##### 新建任务
        新建任务 -> 输入名称 -> 构建一个自由风格的软件项目 -> 确定

    2. ##### 构建触发器和Optional filter参照Java项目部署即可

    3. ##### 构建环境
        ![构建环境](build-node.png)
        `Cache location`选项:
        - 每个节点，这是默认的NPM行为。所有下载包都放在Lunix系统上的~/.npm或Windows系统上的%APP_DATA%\npm-cache中。
        - 每个执行者，每个执行者在~/npm-cache/$executorNumber中都有自己的NPM缓存文件夹。
        - 每个作业，放在$WORKSPACE/npm-cache下的工作区文件夹中。此缓存将与工作区一起被刷除，并且在删除作业时将被删除。

    4. ##### 构建
        ![构建](build-vue.png)
        命令如下:
        ```bash
        # node-sass安装失败可以修改package.json："node-sass": "^4.13.0",并设置淘宝镜像
        # SASS_BINARY_SITE=https://npm.taobao.org/mirrors/node-sass npm install --registry=https://registry.npm.taobao.org
        npm install --registry=https://registry.npm.taobao.org
        npm run build
        cd dist
        tar -zcf ../dist.tar.gz *

        rm -rf /**/dist/*
        tar -zxf /**/dist.tar.gz -C /**/dist
        rm -rf /**/dist.tar.gz
        ```

4. #### 企业微信通知

    1. ##### 安装插件
        [Jenkins企业微信通知插件](https://github.com/Lemon-cxh/cxh-wechat-notification)
    2. ##### 系统配置
        ![系统配置](wechat-notification-configure.png)
    3. ##### 任务单独配置
        ![任务单独配置](wechat-notification-job-configure.png)
    4. ##### 效果图
        ![效果图](wechat-notification-result.jpg)