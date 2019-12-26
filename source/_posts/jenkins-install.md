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

1. ### 安装以及配置Jenkins

    1. #### 安装Jenkins
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

    2. ####  配置Jenkins
        访问<http://localhost:8080>输入上一步的密码，选择第一种安装插件方式即可，之后会提示`无法连接到Jenkins`。
        访问<http://localhost:8080/pluginManager/advanced>找到最底部的Update Site选项，修改URL的https为http。
     
        1. ##### 安装插件
        系统管理 -> 插件管理 -> 可选插件
        搜索安装`Generic Webhook Trigger Plugin`, `NodeJS Plugin`, `Maven Integration plugin`, `Publish Over SSH`

        2. ##### 配置
        - 系统管理 -> 系统设置 -> Publish over SSH
        添加SSH Servers配置，输入Name(此配置名称)，Hostname(服务器地址)， Username(用户名)，Remote Directory(上传文件时的根路径，不填即为用户提供的凭据登录时所在的目录)，如果是密码登录则点击高级勾选`Use password authentication, or use a different key`，在`Passphrase / Password`处填写密码

        - 系统管理 -> 全局工具配置 -> Maven安装以及NodeJS安装
        勾选自动安装即可

        - 用户列表 -> 选择用户 -> 设置 -> API Token
        添加新Token，输入后点击生成并把生成的字符保存起来备用


2. ### 自动部署Gogs上的Java项目
> 目前项目为微服务，使用docker部署，并且只自动部署测试分支，所以以下例子只有推送release分支且指定模块下有修改才触发自动部署。

    1. 新建任务 -> 输入名称 -> 构建一个maven项目 -> 确定 -> 输入源码管理基本信息(不再详细说明)。
    
    2. 构建触发器
        勾选`Generic Webhook Trigger`，`Variable`处输入`repositoryName`,`Expression`处输入`$.repository.name`,`Value filter`处输入`[\s\[\]"]`,勾选`JSONPath`。
        点击新增再添加两个`Variable`,`Expression`和`Value filter`，分别为:`ref`,`$.ref`,`[\s\[\]"]`和`commits`,`$.commits`,`[\s\[\]"]`勾选`JSONPath`。
        `Token`中输入令牌。

    3. Optional filter
        `Expression`处输入`^你的仓库名-refs/heads/release-[\S]*指定的模块路径/`，`Text`处输入`$repositoryName-$ref-$commits`。

    4. Pre Steps
        添加`调用顶层Maven目标`，选择自动安装的maven，`目标`处输入`clean install`。

    5. Build
        `Root POM`处输入指定模块的pom.xml路径，`Goals and options`输入`clean package docker:build`(项目中已配置docker-maven-plugin，镜像直接构建到远程服务器)。

    6. Post Steps
        添加`Send files or execute commands over SSH`，`Name`处选择1.2.2配置中配置好的ssh，`Exec command`处输入`docker stop 容器名;
docker rm 容器名;docker run ******`。

    7. 登录Gogs在仓库管理 -> 管理Web钩子 -> 添加Web钩子 -> gogs，推送地址为：http://用户列表的ID:1.2.2配置中添加的API Token@localhost:8080/generic-webhook-trigger/invoke?token=上面第二步Token中输入的令牌。

3. ### 自动部署Gogs上的Vue项目
    1. 新建任务 -> 输入名称 -> 构建一个自由风格的软件项目 -> 确定 -> 输入源码管理基本信息(不再详细说明)。

    2. 构建触发器
        勾选`Generic Webhook Trigger`，`Variable`处输入`repositoryName`,`Expression`处输入`$.repository.name`，勾选`JSONPath`。
        点击新增再添加一个`Variable`和`Expression`，分别为:`ref`和`$.ref`，勾选`JSONPath`。
        `Token`中输入令牌。

    3. Optional filter
        `Expression`处输入`^(你的仓库名)-(refs/heads/release)$`，`Text`处输入`$repositoryName-$ref`。

    4. 构建环境
        勾选`Provide Node & npm bin/ folder to PATH`，`NodeJS Installation`处选择自动安装的NodeJS，
        `Cache location`选项:
        - 每个节点，这是默认的NPM行为。所有下载包都放在Lunix系统上的~/.npm或Windows系统上的%APP_DATA%\npm-cache中。
        - 每个执行者，每个执行者在~/npm-cache/$executorNumber中都有自己的NPM缓存文件夹。
        - 每个作业，放在$WORKSPACE/npm-cache下的工作区文件夹中。此缓存将与工作区一起被刷除，并且在删除作业时将被删除。

    5. 构建
        添加`执行shell`，命令中输入
        ```bash
        npm i node-sass --sass_binary_site=https://npm.taobao.org/mirrors/node-sass
        npm install --registry=https://registry.npm.taobao.org
        npm run build
        cd dist
        tar -zcf ../dist.tar.gz *
        ```
        添加`Send files or execute commands over SSH`，`Name`处选择1.2.2配置中配置好的ssh，`Source files`处输入`dist.tar.gz`，`Remote directory`处输入上传的目录，如果1.2.2中有配置`Remote Directory`则会在此目录下。`Exec command`处输入
        ```bash
        rm -rf /**/dist/*
        tar -zxf /**/dist.tar.gz -C /**/dist
        rm -rf /**/dist.tar.gz
        ```