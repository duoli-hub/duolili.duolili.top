---
title: SubVersion
date: 2023-11-23 21:48:13
tags:
  - svn
  - Linux
  - 运维
categories:
  - "运维"
  - Linux
keywords: "svn"
description: "CentOS7上搭建基于Apache,http访问的SVN Server;和IF.svnadmin实现web后台可视化管理SVN"
---

# 介绍

CentOS7上搭建基于Apache,http访问的SVN Server;和IF.svnadmin实现web后台可视化管理SVN
iF.SVNAdmin应用程序是您的Subversion授权文件的基于Web的GUI。它基于PHP 5.3，需要安装一个Web服务器（Apache）
该应用程序不需要数据库后端或任何类似的，它完全基于Subversion授权和用户认证文件。（+包含用户和组的LDAP支持）

# 软件准备

- php			5.4.16
- svn			1.7.14
- iF.SVNAdmin	1.6.2

安装apache

```shell
[root@test svn]# yum install httpd -y
## 由于此服务器已使用80端口，所以更改端口为81
	[root@test ~]# vim /etc/httpd/conf/httpd.conf
 	Listen 81
  ServerName localhost:81
```

安装svn服务器（mod_dav_svn是apache服务器访问svn的一个模块）

```shell
[root@test svn]# yum install mod_dav_svn subversion -y
```

验证

```shell
[root@test ~]# httpd --version
[root@test ~]# svnserve --version
[root@test ~]# ls /etc/httpd/modules/ | grep svn
mod_authz_svn.so
mod_dav_svn.so

# 在apache下配置svn
[root@test ~]# vim /etc/httpd/conf.d/subversion.conf
LoadModule dav_svn_module modules/mod_dav_svn.so
LoadModule authz_svn_module modules/mod_authz_svn.so
<Location /svn>
DAV svn
SVNParentPath /var/www/svn   #svn的根目录SSLRequireSSL                #SSL访问权限
AuthType Basic               #Basic认证方式
AuthName "Authorization SVN"   #认证时显示的信息
AuthUserFile /var/www/svn/passwd      #用户文件&密码
AuthzSVNAccessFile /var/www/svn/authz  #访问权限控制文件
Require valid-user            #要求真实用户,不能匿名
</Location>
```

# 建立SVN Server仓库

通过如下命令建立svn仓库
其中/var/www/svn是准备放仓库的目录,这个目录可以放置多个代码仓库

```shell
[root@test ~]# mkdir -p /var/www/svn
[root@test ~]# svnadmin create /var/www/svn/sungeek
[root@test ~]# ls /var/www/svn/sungeek
conf  db  format  hooks  locks  README.txt
[root@test ~]# chown -R apache.apache /var/www/svn

# 创建用户文件passwd和权限控制文件authz
[root@test ~]# touch /var/www/svn/passwd
[root@test ~]# touch /var/www/svn/authz
```

# 配置安装PHP&IF.SVNadmin

由于iF.SVNAdmin使用php写的，因此我们需要安装php

```shell
[root@test ~]# yum install php -y
```

安装配置if.svnadmin

```shell
[root@test ~]# yum install wget unzip -y
[root@test ~]# cd /usr/src/
[root@test src]# wget http://sourceforge.net/projects/ifsvnadmin/files/svnadmin-1.6.2.zip --no-check-certificate
[root@test src]# unzip svnadmin-1.6.2.zip
[root@test src]# cp -r iF.SVNAdmin-stable-1.6.2 /var/www/html/svnadmin
[root@test src]# cd /var/www/html
[root@test html]# chown -R apache.apache svnadmin
[root@test html]# cd /var/www/html/svnadmin
[root@test svnadmin]# chmod -R 777 data
[root@test ~]# chown -R apache.apache /var/www/svn/passwd
[root@test ~]# chown -R apache.apache /var/www/svn/authz
```

# 启动服务

> 如果开启了防火墙, 需要开启httpd访问权限
> firewall-cmd --permanent --add-service=http 
> firewall-cmd --permanent --add-service=https 
> firewall-cmd --reload　

修改配置文件

```shell
[root@test ~]# vim /etc/sysconfig/svnserve
OPTIONS="-r /var/www/svn"
[root@test ~]# systemctl start httpd
[root@test ~]# systemctl enable httpd
```

# Web配置

浏览器访问：[http://192.168.1.20:81/svnadmin](http://192.168.1.20:81/svnadmin)
配置如下图（输入每个配置信息可以点击旁边的Test测试是否输入正确，最后保存配置）
![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/SubVersion/1.png)

![image.png](https://cdn.jsdelivr.net/gh/duoli-hub/cdn/source/img/markdown/SubVersion/2.png)