---
title: Docker Compose 的安装
date: 2024-09-20 11:22:40
tags:
  - docker compose
  - 运维
categories:
  - "运维"
  - docker
keywords: "docker"
description: 安装 Docker Compose

---



# 1. 安装 Docker

```shell
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker && systemctl enable docker
```

# 2. 下载 docker-compose

```shell
curl -L "https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 安装指定版本：将2.16.0更换至指定版本号即可

# 默认在github下载，国内网络可能非常慢，可以尝试使用以下进行下载
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.16.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

# 3. 赋权+软连接

```shell
chmod +x /usr/local/bin/docker-compose
ln -sv /usr/local/bin/docker-compose /usr/bin/docker-compose
```

# 4. 测试

```shell
docker-compose version
```



**注意版本对应关系**：[点击跳转](https://docs.docker.com/compose/compose-file/compose-versioning/)



# 5. 扩展：一键安装脚本

只针对 CentOS 系统

```shell
$ cat install-docker-compose.sh
#!/bin/bash
# install docker
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker && systemctl enable docker

# install docker-compose
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.16.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -sv /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose version
```



