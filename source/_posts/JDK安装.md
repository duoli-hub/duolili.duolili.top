---
title: JDK安装
date: 2023-11-23 21:32:08
tags:
  - JDK
  - Linux
  - 运维
categories:
  - "运维"
  - Linux
keywords: "JDK"
description: JDK安装教程
---

# OpenJDK11

## 二进制安装

- 国内华为镜像下载：[https://repo.huaweicloud.com/java/jdk/11.0.2+9/](https://repo.huaweicloud.com/java/jdk/11.0.2+9/)
- injdk 镜像下载：[https://www.injdk.cn/](https://www.injdk.cn/)
- 官网下载：[https://www.oracle.com/java/technologies/downloads/#java11](https://www.oracle.com/java/technologies/downloads/#java11) 【需Oracle账号：1004361678@qq.com/CloudEasy@2020】

下载二进制包并上传服务器至/usr/local/jdk/路径下，并解压

```shell
[root@test jdk]# tar -zxvf jdk-11.0.2_linux-x64_bin.tar.gz
```

编辑环境变量

```shell
[root@test jdk]# vim /etc/profile.d/java.sh
export JAVA_HOME=/usr/local/jdk/jdk-11.0.2
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
[root@test jdk]# source /etc/profile.d/java.sh
```

验证

```shell
[root@test jdk]# java -version
java version "11.0.2" 2019-01-15 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.2+9-LTS)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.2+9-LTS, mixed mode)
```

## yum安装

```shell
yum remove java
yum search openjdk
yum install java-11-openjdk java-11-openjdk-devel -y
java -version
```