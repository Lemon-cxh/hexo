---
title: Jenkins + Gogs 自动化部署
date: 2019-09-29 16:53:54
categories: 
- Jenkins
tags:
- Jenkins
description: 使用Docker运行Jenkins,结合Gogs完成Java和Vue项目的自动化部署
---
> 使用Docker运行Jenkins,结合Gogs完成Java和Vue项目的自动化部署
> Docker的安装可以参考{% post_link docker-deploy-springCloud %}

1. #### 安装以及配置Jenkins

    1. ##### 安装Jenkins
        添加用户
        ```bash
        useradd -u 1000 jenkins
        ```
        新建目录
        ```bash
        mkdir /var/jenkins_home
        ```
        修改所属用户
        ```bash
        chown jenkins /var/jenkins_home
        ```
        运行容器
        ```bash
        docker run -d -i --name jenkins --network docker-net -p 8080:8080 -p 50000:50000 -v /var/jenkins_home:/var/jenkins_home -v /etc/localtime:/etc/localtime:ro jenkins/jenkins:lts
        ```
        查看日志 会实时显示日志，其中类似f3f67320f92e4ba6bdecb083a03af92c的一行即为密码
        ```bash
        docker logs -f jenkins
        ```

    2. #####  配置Jenkins
        访问<http://localhost:8080>输入上一步的密码，选择第一种安装插件方式即可，之后会提示`无法连接到Jenkins`。
        访问<http://localhost:8080/pluginManager/advanced>找到最底部的Update Site选项，修改URL的https为http。
     
        1. ###### 安装插件
        系统管理 -> 插件管理 -> 可选插件
        搜索安装`Generic Webhook Trigger Plugin`, `NodeJS Plugin`, `Maven Integration plugin`, `Publish Over SSH`

        2. ###### 配置
        - 系统管理 -> 系统设置 -> 新增Publish over SSH配置
        ![SSH Servers配置](ssh-config.png)

        - 系统管理 -> 全局工具配置 -> Maven安装以及NodeJS安装,勾选自动安装即可
        ![SSH Servers配置](tool-config.png)

        - 用户列表 -> 选择用户 -> 设置 -> API Token
        添加新Token，输入后点击生成并把生成的字符保存起来备用


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

        推送地址为：http://`用户列表的ID`:`用户设置中添加的API Token`@localhost:8080/generic-webhook-trigger/invoke?token=`构建触发器中输入的Token`
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
        npm install --registry=https://registry.npm.taobao.org
        npm run build
        cd dist
        tar -zcf ../dist.tar.gz *

        rm -rf /**/dist/*
        tar -zxf /**/dist.tar.gz -C /**/dist
        rm -rf /**/dist.tar.gz
        ```