---
title: Docker运行Arcsoft人脸识别服务
tags:
  - Docker
keywords: Arcsoft,Docker
categories:
  - Docker
description: Docker 运行 Arcsoft 人脸识别服务，SDK升级后需要GLIBCXX_3.4.21、GLIBC_2.18，所以自行构建基础镜像
date: 2021-11-03 16:54:34
---


#### 服务DockerFile配置

项目中 `arcsoft` 目录中存放着算法的三个.so文件，以下为项目的 DockerFile：

```DockerFile
FROM centos7_jdk8:1
VOLUME /tmp
COPY arcsoft/ /usr/lib/
ADD iot-arcsoft.jar iot-arcsoft.jar
ENV LD_LIBRARY_PATH=/usr/lib
RUN bash -c 'touch /iot-arcsoft.jar' \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-Dfile.encoding=utf-8","-jar","iot-arcsoft.jar"]
```

#### 基础镜像构建

由于SDK更新，原先基于`store/oracle/serverjre:8`构建的镜像无法满足要求，
运行报错。

- 提示 GLIBCXX_3.4.21 not found 

  查看系统支持的版本
  ```bash
  strings /usr/lib64/libstdc++.so.6 | grep GLIBCXX* 
  ```

- 提示 GLIBC_2.18 not found
  
  查看系统支持的版本
  ```bash
  strings /lib64/libc.so.6 | grep GLIBC_*
  ```


提前准备好三个文件，和 DockerFile 处于同一个目录。
- [jdk-8u311-linux-x64.tar.gz](https://www.oracle.com/java/technologies/downloads/#java8)
- [glibc-2.18.tar.gz](https://ftp.gnu.org/gnu/libc/glibc-2.18.tar.gz)
- libstdc++.so.6.0.25

> `libstdc++.so.6.0.25` 文件在服务器中寻找到的
> find / -name "libstdc++.so.6*" 查看高版本的lib库


```DockerFile
FROM centos:centos7

ADD jdk-8u311-linux-x64.tar.gz /opt
ADD glibc-2.18.tar.gz /opt
ADD libstdc++.so.6.0.25 /usr/lib64/

RUN cd / && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
  && yum makecache \
  && yum install -y wget aclocal automake autoconf make gcc gcc-c++ glibc \
  && yum clean all

RUN cd /opt/glibc-2.18 && mkdir build && cd build \
  && ../configure --prefix=/usr && make -j4 && make install \
  && cd /usr/lib64 && ln -sf /usr/lib64/libstdc++.so.6.0.25 /usr/lib64/libstdc++.so.6
 
WORKDIR /opt

ENV JAVA_HOME=/opt/jdk1.8.0_311
ENV JRE_HOME=${JAVA_HOME}/jre
ENV CLASSPATH=.:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar:${JRE_HOME}/lib/dt.jar
ENV PATH=${JAVA_HOME}/bin:${PATH}:${JRE_HOME}/bin
 
CMD ["/bin/bash"]
```

#### 升级GCC

如果服务器中没有高版本的`libstdc++.so.6*`文件，需要安装高版本[gcc-6.5.0](https://ftp.gnu.org/gnu/gcc/gcc-6.5.0/gcc-6.5.0.tar.gz)(可能需要安装libstdc++、libstdc++-devel、bzip2)，下载并解压到`/opt`目录下。在`/opt`目录下按顺序执行
```bash
./contrib/download_prerequisites

mkdir build && cd build

../configure --enable-checking=release --enable-languages=c,c++ --disable-multilib
 
make -j4

make install
```
然后查找高版本libstdc++.so.6文件，找到后回到上一步基础镜像构建
```bash
find / -name "libstdc++.so.6*"
```