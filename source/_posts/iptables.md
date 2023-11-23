---
title: iptables
date: 2023-11-23 16:44:45
tags:
  - iptable
  - Linux
  - 运维
categories:
  - "运维"
  - Linux
keywords: "iptable"
description: iptables入门
---





## 四表五链

- 四表

1. FILTER：进行数据包的过滤性处理
   1. INPUT链：对需要进入的流量数据进行规则匹配（外部-->本机）
   2. OUTPUT链：对需要出去的流量进行规则匹配（本机-->外部）
   3. FORWARD：对流经的流量数据进行规则匹配（外部1-->本机转发-->外部2）
2. NAT表：对数据包中的地址信息进行转换
   1. postrouting：对某数据处理之后要做的事情即对经过路由之后的数据的处理性操作，实现内部共享上网功能
   2. prerouting：对某数据处理之前要做的事情即对经过路由之前的数据的处理性操作，实现外网转发内网功能
3. MANGLE表：对数据包信息进行标记修改操作
4. RAW表：将数据包中的信息进行拆解





## 安装防火墙服务端

````shell
yum install iptables-services -y
````

## 启动

```sh
systemctl start iptables
```

## 查看规则

```shell
iptables -nL
```

## 防火墙初始化规则

```shell
# 清空所有链上的规则
iptables -F
# 清空所有计数器信息(计数器：帮助排查规则不通的问题的)
iptabled -Z
# 清空用户自定义链的所有规则
iptables -X
```

## 查看防火墙规则

> -L	通过列表显示当前防火墙的规则信息
>
> -n	默认按照ip与端口显示，不解析协议名称
>
> -v	详细显示规则信息
>
> --line-numbers	按照序号显示规则
>
> -t	指定显示规则的表（不指定时，默认显示FILTER表的规则）

```shell
# 查看nat表的所有规则
iptables -t nat -nL
```

## 基于filter表规则配置

> 				-t			指定表，不指定时默认为filter表
> 				-A			在指定链添加规则，默认添加至末尾
> 				-I			在指定链中插入规则，默认添加至首行
> 				-p			指定协议：TCP、UDP、ICMP、ARP、HTTP、FTP、DNS
> 				--dport		指定目的端口
> 				--sport		指定来源端口
> 				-s			指定来源地址信息
> 				-i			指定接收流向的网卡设备
> 				-o			指定出口流量的网卡设备
> 				-j			指定规则策略：DROP(丢弃包)、ACCEPT、REJECT(拒绝)
> 				!			取反

- 阻止所有用户不能访问ssh远程服务

```shell
# 操作filter表
iptables -A INPUT -p tcp --dport 22 -j DROP
```

- 阻止windows远程连接到虚拟机，允许其他网段连接

```shell
# 操作filter表
iptables -A INPUT -p tcp -s 10.0.0.1 --dport 22 -j DROP
```

- 只允许windows连接，其他网段全不允许

```shell
# 方法一
iptables -A INPUT -p tcp -s 10.0.0.1 --dport 22 -j ACCEPT
iptables -A INPUT -p tcp -s 0.0.0.0/0 --dport 22 -j DROP
# 方法二
iptables -A INPUT -p tcp ! -s 10.0.0.1 --dport 22 -j DROP
```

- 只允许通过内网访问80端口，不允许通过外网访问80端口

```shell
# 两张网卡的情况下
iptables -A INPUT -p tcp -i eth1 --dport 80 -j ACCEPT
iptables -A INPUT -p tcp -i eth0 --dport 80 -j DROP
```

### 定义防火墙规则顺序

> -I			在指定链的某个规则前插入规则

- 凡是访问本机80端口，都放行，其他端口都拒绝

```shell
# 放行所有本地windows请求
iptables -A INPUT -p tcp -s 10.0.0.1 -j ACCEPT
# 拒绝所有tcp请求
iptables -A INPUT -p tcp -j DROP
# 查看规则序号
iptables -nL --line-numbers
# 在第二条规则前插入一条规则：放通所有请求80端口的请求
iptables -I INPUT 2 -p tcp --dport 80 -j ACCEPT
```

### 基于现有规则去修改

> -R		修改规则

- 修改第二条规则：将原放通80端口修改为放通22端口

```shell
iptables -R INPUT 2 -p tcp --dport 22 -j ACCEPT
```

### 同时放行某范围内的所有端口

```shell
iptables -R INPUT 2 -p tcp --dport 22:80 -j ACCEPT
```

### 指定开放某几个端口

> -m		指定某些扩展模块，实现某些功能
>
> multiport		模块：用于指定多个端口进行规则匹配
>
> limit			模块：设定单位时间内访问多少个ip（实现防护恶意攻击、恶意点击行为）
>
> --limit-burst		模块：设定阈值，到达第几个开始触发规则

```shell
# 同时放通22、80、443端口
iptables -R INPUT 2 -p tcp -m multiport --dport 22,80,443 -j ACCEPT
```

### 禁用ping

ping依赖的是icmp协议

icmp基于ping-pong，ping是icmp其中的一个请求类型：0 回复	8 请求

```shell
# 方法一：基于INPUT链
iptables -A INPUT -p icmp --icmp-type 8 -j DROP
# 方法二：基于OUTPUT链
iptables -A OUTPUT -p icmp --icmp-type 0 -j DROP
```

### 限制每分钟最多可以ping6次，从第十次开始触发

```shell
iptables -A OUTPUT -p icmp --icmp-type 0 -j DROP
iptables -I OUTPUT 1 -p icmp --icmp-type 0 -m limit 6/min --limit-burst 10 -j ACCEPT
```

### 删除规则

> -D

```shell
# 方法一
iptables -D OUTPUT -p icmp --icmp-type 0 -j DROP
# 方法二：基于序号删除
iptables -D INPUT 2
```

### 修改默认规则

默认规则放通全部

修改默认规则拒绝全部

```shell
iptables -P INPUT DROP
```

## 基于nat表规则配置

`POSTROUTING`：地址或端口映射。先进行路由，后进行地址转换

`PREROUTING`：地址或端口映射。先进行地址转换，后进行路由转发

### NAT实现内部网络共享上网功能

实现demo2通过demo1的ens33网卡访问百度

- 环境

  demo1：

  ​	ens33：10.0.0.31	可通外网

  ​	ens34：192.168.10.10	内网

  demo2：

  ​	ens34:192.168.10.11	内网

- 操作

1. demo2配置内网网卡网关为防火墙的内网ip，并配置DNS

```shell
# demo2
vim /etc/sysconfig/network-scripts/ifcfg-ens33
GATEWAY=192.168.10.10
DNS1=114.114.114.114
systemctl restart network		## 或者停止网卡再启动 ifdown ens34 && ifup ens34
```

2. demo1开启内核转发参数

```shell
vim /etc/sysctl.conf
net.ipv4.ip_forward=1
# 加载内核参数
sysctl -p
```

3. 配置防火墙开启内网转发

   > -o		指定流量包的出口设备网卡是外网网卡
   >
   > -j SNAT	规则策略是将请求包的来源地址进行转换
   >
   > --to-source		转换为的目标ip是多少

```shell
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o ens33 -j SNAT --to-source 10.0.0.31
# 或者写成以下亦可
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o ens33 -j MASQUERADE
```

4. 查看防火墙规则

```shell
iptables -t nat -nL
```

5. demo2 访问百度测试

```shell
ping www.baidu.com
```

