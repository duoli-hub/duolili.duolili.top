---
title: Ansible-Semaphore
date: 2023-11-23 21:41:08
tags:
  - ansible
  - Linux
  - 运维
categories:
  - "运维"
  - Linux
keywords: 
  - "ansible"
  - "ansible web"
description: "一个 ansible web 界面，通过直观的操作和可视化界面工具，使得运行ansible playbook变的简单而有效"
---

一个 ansible web 界面，通过直观的操作和可视化界面工具，使得运行ansible playbook变的简单而有效
默认端口 3000

参考

- Github：[点击跳转](https://github.com/ansible-semaphore/semaphore)
- 官方文档：[点击跳转](https://docs.ansible-semaphore.com/administration-guide/installation)

# 安装

按照安装方式的不同，可分为4种：snap（snap命令行安装）、package manage（包管理器安装）r、docker（容器化安装）、binary file（二进制安装）

## Snap

环境准备
Centos7.6 1810	192.168.1.20

安装snap命令

```shell
[root@test ~]# yum install epel-release -y
[root@test ~]# yum install snapd -y
[root@test ~]# systemctl start snapd.socket		# snapd的socket用于与命令snap通信
[root@test ~]# systemctl start snapd.service
[root@test ~]# ln -sf /var/lib/snapd/snap /snap
[root@test ~]# snap version
```

安装semaphorfe

```shell
[root@test ~]# snap install semaphore
# 此时可通过3000端口打开ui页面，但无法进行登录，因此，需要设置登录用户
```

![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/Ansible-Semaphore/1.png)
设置管理员登陆用户

```shell
[root@test ~]# snap stop semaphore
[root@test ~]# /snap/bin/semaphore user add --admin --login duoli --name=duoli --email=duoli@163.com --password=12345
[root@test ~]# snap start semaphore
```

此时可以通过duoli/12345进行登录
![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/Ansible-Semaphore/2.png)
![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/Ansible-Semaphore/3.png)

snap命令扩展

```shell
# 查看semaphore服务状态
[root@test ~]# snap services semaphore
Service               Startup  Current  Notes
semaphore.semaphored  enabled  active   -

# 查看服务配置信息
[root@test ~]# snap get semaphore

# 设置服务配置【可配置信息参考：https://docs.ansible-semaphore.com/administration-guide/configuration】
[root@test ~]# snap set semaphore port=4444
[root@test ~]# snap restart semaphore
```

## Package Manage

## Docker

## Binary File

创建用户

```shell
[root@test ~]# useradd semaphore
```

安装ansible

```shell
[root@test ~]# yum install epel-release -y
[root@test ~]# yum install ansible -y
[root@test ~]# ansible --version
```

安装python3

```shell
# 安装编译工具
[root@test ~]# yum -y groupinstall "Development tools" 
[root@test ~]# yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
[root@test ~]# yum install -y libffi-devel zlib1g-dev
[root@test ~]# yum install zlib* -y
# 下载安装包并解压
[root@test ~]# wget https://www.python.org/ftp/python/3.7.2/Python-3.7.2.tar.xz
[root@test ~]# tar -xvJf  Python-3.7.2.tar.xz
# 编译安装
[root@test ~]# cd Python-3.7.2
[root@test Python-3.7.2]# mkdir -p /usr/local/python3
[root@test Python-3.7.2]# ./configure --prefix=/usr/local/python3 --enable-optimizations --with-ssl
	#第一个指定安装的路径,不指定的话,安装过程中可能软件所需要的文件复制到其他不同目录,删除软件很不方便,复制软件也不方便.
	#第二个可以提高python10%-20%代码运行速度.
	#第三个是为了安装pip需要用到ssl
[root@test Python-3.7.2]# make && make install
[root@test Python-3.7.2]# ln -s /usr/local/python3/bin/python3 /usr/local/bin/python3
[root@test Python-3.7.2]# ln -s /usr/local/python3/bin/pip3 /usr/local/bin/pip3
[root@test Python-3.7.2]# python3 -V
[root@test Python-3.7.2]# pip3 -V
```

创建配置文件

```shell
[root@test ~]# su - semaphore
[semaphore@test ~]$ vim requirements.txt
ansible
# for common jinja-filters
netaddr
jmespath
# for common modules
pywinrm
passlib
requests
docker



[semaphore@test ~]$ vim requirements.yml
---
collections:
  - 'namespace.collection'
  # for common collections:
  - 'community.general'
  - 'ansible.posix'
  - 'community.mysql'
  - 'community.crypto'

roles:
  - src: 'namespace.role'
```

安装mysql

```shell
[root@test ~]# vim mysql.sh
#!/bin/bash

yum remove `rpm -qa| grep mariadb` -y
yum install wget -y
#wget http://aliyun-ops.oss-cn-beijing.aliyuncs.com/software/mysql-5.7.32-linux-glibc2.12-x86_64.tar.gz
#tar -xf mysql-5.7.32-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
tar -xf mysql-5.7.38-el7-x86_64.tar.gz -C /usr/local/
cd /usr/local/ && ln -sv mysql-5.7.38-el7-x86_64 mysql
useradd -s /sbin/nologin mysql
chown -R mysql.mysql mysql && chown -R mysql.mysql mysql-5.7.38-el7-x86_64
echo -e 'export MYSQL_HOME=/usr/local/mysql\nexport PATH=$PATH:$MYSQL_HOME/bin' >>  /etc/profile.d/mysql.sh
mkdir -p /data/mysql/data
chown -R mysql.mysql /data/mysql/data
/usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/loca/mysql --datadir=/data/mysql/data
cat >/etc/my.cnf <<EOF
[mysqld]
skip_ssl
bind-address=0.0.0.0
port=3306
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysql/data
socket=/tmp/mysql.sock
symbolic-links=0
#!includedir /etc/my.cnf.d
init_connect='SET NAMES utf8'
skip-character-set-client-handshake
max_allowed_packet = 34M
innodb_log_file_size = 256M
transaction-isolation=READ-COMMITTED
default-storage-engine=INNODB
character_set_server=utf8mb4
innodb_default_row_format=DYNAMIC
innodb_large_prefix=ON
innodb_file_format=Barracuda
innodb_log_file_size=2G
EOF

chown -R mysql.mysql /etc/my.cnf

cat >/etc/systemd/system/mysqld.service <<EOF
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf
LimitNOFILE=5000
EOF

systemctl start mysqld && systemctl enable mysqld
#/usr/local/mysql/bin/mysqladmin -hlocalhost -uroot password CloudEasy@2020
#/usr/local/mysql/bin/mysql -uroot -pCloudEasy@2020 -NBe "update mysql.user set host = '%' where user = 'root';"
#/usr/local/mysql/bin/mysql -uroot -pCloudEasy@2020 -NBe "flush privileges;"
#reboot


cat > /tmp/init.sh <<EOF
#!/bin/bash
mysqladmin -hlocalhost -uroot password CloudEasy@2020
mysql -uroot -pCloudEasy@2020 -NBe "update mysql.user set host = '%' where user = 'root';"
mysql -uroot -pCloudEasy@2020 -NBe "flush privileges;"
echo -e "\033[31mROOT密码：\033[33mCloudEasy@2020\033[0m"
EOF

echo -e "\033[31m请按顺序执行：\n\033[31m第一步：\033[32msource /etc/profile.d/mysql.sh\n\033[31m第二步：\033[32msh /tmp/init.sh\n\033[31m提醒：\033[31m若执行报错，从第一步重新执行\033[0m"

[root@test ~]# sh mysql.sh
[root@test ~]# source /etc/profile.d/mysql.sh
[root@test ~]# sh /tmp/init.sh
[root@test ~]# mysql -uroot -pCloudEasy@2020 -NBe "create database semaphore;"
```

安装semaphore

```shell
[root@test ~]# su - semaphore
[semaphore@test ~]$ wget https://github.com/ansible-semaphore/semaphore/releases/download/v2.8.75/semaphore_2.8.75_linux_amd64.tar.gz
[semaphore@test ~]$ tar xf semaphore_2.8.75_linux_amd64.tar.gz
[semaphore@test ~]$ ./semaphore setup		# 设置配置，会生成config.json文件
###验证运行：【nohup ./semaphore server --config /home/semaphore/config.json &】
[semaphore@test ~]$ exit
[root@test ~]# ln -sv /home/semaphore/semaphore /usr/bin/semaphore
[root@test ~]#  cat > /etc/systemd/system/semaphore.service <<EOF
[Unit]
Description=Ansible Semaphore
Documentation=https://docs.ansible-semaphore.com/
Wants=network-online.target
After=network-online.target
ConditionPathExists=/usr/bin/semaphore
ConditionPathExists=/home/semaphore/config.json

[Service]
User=semaphore
Group=semaphore
Restart=always
RestartSec=10s
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:~/.local/bin"

#ExecStartPre=/bin/bash -c 'ansible-galaxy collection install --upgrade -r /home/semaphore/requirements.yml'
#ExecStartPre=/bin/bash -c 'ansible-galaxy role install --force -r /home/semaphore/requirements.yml'
#ExecStartPre=/bin/bash -c 'python3 -m pip3 install --upgrade --user -r /home/semaphore/requirements.txt'

ExecStart=/usr/bin/semaphore service --config /home/semaphore/config.json
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF

### 实验时，配置上下文时启动始终有问题
[root@test ~]# systemctl daemon-reload
[root@test ~]# systemctl start semaphore
[root@test ~]# systemctl enbale semaphore
```


# 使用

首次登录需要配置项目名称以及并行任务数量（0表示不限制）
![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/Ansible-Semaphore/4.png)
![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/Ansible-Semaphore/5.png)