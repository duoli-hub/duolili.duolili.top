---
title: MySQL基础
date: 2023-12-01 00:23:29
tags:
  - MySQL
categories:
  - [运维, 数据库, MySQL]
keywords: "mysql"
description: MySQL入门
---

# 

# 1. 数据库分类

RDBMS ： Oracle ，MySQL ，PG，MSSQL 

NoSQL ： MongoDB ，Redis ，ES 

NEWSQL （分布式）： TiDB，Spanner ，AliSQL ，OB ，PolarDB 

关系型数据库的特点 

- 1.数据以表格的形式出现 
- 2.每⾏为各种记录名称 
- 3.每列为记录名称所对应的数据域 
- 4.许多的⾏和列组成⼀张表单 
- 5.若⼲的表单组成database



# 2. 安装MySQL

1. **删除默认MariaDB并添加mysql用户**

```shell
[root@node01 ~]# yum remove `rpm -qa| grep mariadb` -y
[root@node01 ~]# useradd -s /sbin/nologin mysql
[root@node01 ~]# yum install libaio -y
```

2. **下载软件包并解压**

官网下载：https://downloads.mysql.com/archives/community/

```shell
[root@node01 ~]# wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz
[root@node01 ~]# tar -xf mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
[root@node01 ~]# ln -sv /usr/local/mysql-5.7.43-linux-glibc2.12-x86_64 /usr/local/mysql
```

3. **创建数据目录并授权**

```shell
[root@node01 ~]# mkdir -p /data/mysql/data && mkdir -p /data/mysql/binlog
[root@node01 ~]# chown -R mysql.mysql /usr/local/mysql-5.7.43-linux-glibc2.12-x86_64 && chown -R mysql.mysql /usr/local/mysql && chown -R mysql.mysql /data/mysql/data && chown -R mysql.mysql /data/mysql/binlog
```

4. **初始化系统数据**

```shell
[root@node01 ~]# /usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/loca/mysql --datadir=/data/mysql/data
```

5. **创建配置文件**

```shell
[root@node01 ~]# cat >/etc/my.cnf <<EOF
[mysqld]
skip_ssl
bind-address=0.0.0.0
server_id=6
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

[root@node01 ~]# chown -R mysql.mysql /etc/my.cnf
```

6. **启动**

```shell
[root@node01 ~]# cat >/etc/systemd/system/mysqld.service <<EOF
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

[root@node01 ~]# systemctl start mysqld && systemctl enable mysqld
```

7. **设置环境变量**

```shell
[root@node01 ~]# echo -e 'export MYSQL_HOME=/usr/local/mysql\nexport PATH=$PATH:$MYSQL_HOME/bin' >>  /etc/profile.d/mysql.sh
[root@node01 ~]# source /etc/profile.d/mysql.sh
```

8. **初始化配置**

```shell
# 设置root密码，默认为空
[root@node01 ~]# mysqladmin -hlocalhost -uroot password CloudEasy@2020
# 设置root用户允许远程登陆（可选）
[root@node01 ~]# mysql -uroot -pCloudEasy@2020 -NBe "update mysql.user set host = '%' where user = 'root';"
# 刷新
[root@node01 ~]# mysql -uroot -pCloudEasy@2020 -NBe "flush privileges;"
```

9. **远程连接验证**

10. **扩展：忘记管理员用户密码**

- 关闭数据库
- 启动数据库到维护模式

```shell
mysqld_safe --skip-grant-tables --skip-networking &
```

- 登陆并修改密码

```shell
mysql
flush privileges;
alter user root@'%' identified by '1';
```

- 关闭数据库，正常启动并进行验证



# 3. 二进制日志 binlog

## 3.1 日志管理

查看日志路径

```mysql
select @@datadir;		# 查看数据存储路径
select @@log_error;		# 查看错误日志路径
```

修改错误日志路径

```shell
vim /etc/my.cnf
log_error=/data/mysql/data/mysql.err
```

## 3.2 二进制日志（binlog）

作用：数据恢复，主从复制

注意：

- 尽量和数据盘分开放，重新挂载一块盘
- binlog默认不开启

**3.2.1 binlog记录的内容**

记录了数据库中所有变更类的操作：DDL、DCL、DML

细节：

- 对于DDL和DCL语句，记录发生过的语句
- DML（IUD）
  - 记录格式：
    - ROW（5.7默认）：RBR 行记录模式 ---> 记录的是行的变化
    - STATEMENT：SBR 语句记录模式 ---> 记录操作语句
    - MIXED：MBR 混合记录模式 ---> 由MySQL去判断决定用RBR或SBR

优缺点：

- RBR：逐行记录日志，日志量很大，可读性差，但是够严禁，不会出现记录错误
- SBR：只记录语句本身，日志量很少，可读性强，但对于函数类的操作，将来恢复会造成错误

查看默认记录格式

```mysql
mysql> select @@binlog_format;
+-----------------+
| @@binlog_format |
+-----------------+
| ROW             |
+-----------------+
1 row in set (0.00 sec)
```

自定义格式

```shell
vim /etc/my.cnf
binlog-format="ROW"
```



**3.2.2 开启binlog**

```shell
[root@node01 ~]# systemctl stop mysqld
[root@node01 ~]# vim /etc/my.cnf
log_bin=/data/mysql/binlog/mysql-bin
[root@node01 ~]# mkdir -p /data/mysql/binlog
[root@node01 ~]# chown -R mysql.mysql /data/mysql
[root@node01 ~]# systemctl start mysqld
```

PS：开启实际上就是指定存放位置

**3.2.3 查看binlog信息**

```mysql
mysql> show variables like '%log_bin%';
+---------------------------------+------------------------------------+
| Variable_name                   | Value                              |
+---------------------------------+------------------------------------+
| log_bin                         | ON                                 |
| log_bin_basename                | /data/mysql/binlog/mysql-bin       |
| log_bin_index                   | /data/mysql/binlog/mysql-bin.index |
| log_bin_trust_function_creators | OFF                                |
| log_bin_use_v1_row_events       | OFF                                |
| sql_log_bin                     | ON                                 |
+---------------------------------+------------------------------------+
6 rows in set (0.01 sec)
```



**3.2.4 二进制日志事件（event）**

二进制日志的最小记录单元：event

对于DDL和DCL：一个语句就是一个event

对于DML：只记录已提交的事务

例如：

```
position号码
begin; 154 - 340
DML1 340 - 460
DML2 460 - 550
commit; 550 - 760
```



event的组成：三部分组成

（1）事件的开始标识（2）事件内容（3）事件的结束标识

Position：某个事件在binlog中的相对位置号（位置号是为了方便截取事件）

开始标识：at 200

结束标识：end_log_pos 340





**3.2.5 查看二进制日志**

查看binlog在本地的存储路径

```mysql
mysql> show variables like '%log_bin%';
+---------------------------------+------------------------------------+
| Variable_name                   | Value                              |
+---------------------------------+------------------------------------+
| log_bin                         | ON                                 |
| log_bin_basename                | /data/mysql/binlog/mysql-bin       |
| log_bin_index                   | /data/mysql/binlog/mysql-bin.index |
| log_bin_trust_function_creators | OFF                                |
| log_bin_use_v1_row_events       | OFF                                |
| sql_log_bin                     | ON                                 |
+---------------------------------+------------------------------------+
6 rows in set (0.01 sec)
```

查看本地存储文件

```shell
[root@node01 ~]# ll /data/mysql/binlog/
total 8
-rw-r----- 1 mysql mysql 154 Nov 25 14:13 mysql-bin.000001
-rw-r----- 1 mysql mysql  36 Nov 25 14:13 mysql-bin.index
```

查看正在使用哪个日志

​	PS：posttion 号154 之前是系统记录的版本信息，如果版本不一致不能进行数据恢复

- 方式一：show binary logs;

```mysql
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       154 |
+------------------+-----------+
1 row in set (0.00 sec)

/* 
通常最后一条就是当前正在使用的日志 
Log_name : 目前MySQL存在的二进制日志名
File_size : 目前MySQL用到哪个position号
*/
```

- 方式二：show master status;

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```



**3.2.6 查看二进制文件内容**

1. 获取二进制文件名

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

2. 查看事件内容：每一行都是一个事件

- 方法一：在mysql内部查看

```mysql
mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         6 |         123 | Server ver: 5.7.43-log, Binlog ver: 4 |
| mysql-bin.000001 | 123 | Previous_gtids |         6 |         154 |                                       |
| mysql-bin.000001 | 154 | Anonymous_Gtid |         6 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
| mysql-bin.000001 | 219 | Query          |         6 |         326 | create database test charset=utf8     |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
4 rows in set (0.00 sec)
```

```ruby
Log_name ：日志名 
Pos ：事件开始的position 
Event_type ：事件类型 
Server_id ：发生在哪台机器的事件 
End_log_pos 事件结束的位置号 
Info ：事件内容
```



- 方法二：在mysql外部查看---解析为txt文件

```shell
[root@node01 ~]# mysqlbinlog /data/mysql/binlog/mysql-bin.000001 | grep -v 'SET' > /tmp/a.txt
[root@node01 ~]# cat /tmp/a.txt
DELIMITER /*!*/;
# at 4
#231125 14:13:53 server id 6  end_log_pos 123 CRC32 0x1d5a6ecc  Start: binlog v 4, server v 5.7.43-log created 231125 14:13:53 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
BINLOG '
oZBhZQ8GAAAAdwAAAHsAAAABAAQANS43LjQzLWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAChkGFlEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AcxuWh0=
'/*!*/;
# at 123
#231125 14:13:53 server id 6  end_log_pos 154 CRC32 0x3fc1f191  Previous-GTIDs
# [empty]
# at 154
#231125 15:04:44 server id 6  end_log_pos 219 CRC32 0x676b3f18  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no
# at 219
#231125 15:04:44 server id 6  end_log_pos 326 CRC32 0x8cc2c541  Query   thread_id=3     exec_time=0     error_code=0
/*!\C utf8mb4 *//*!*/;
create database test charset=utf8
/*!*/;
DELIMITER ;
# End of log file
```

- 方法二：解析日志文件参数：--base64-output=decode-rows

```shell
[root@node01 ~]# mysqlbinlog --base64-output=decode-rows -vvv /data/mysql/binlog/mysql-bin.000001 > /tmp/b.txt
[root@node01 ~]# cat /tmp/b.txt
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#231125 14:13:53 server id 6  end_log_pos 123 CRC32 0x1d5a6ecc  Start: binlog v 4, server v 5.7.43-log created 231125 14:13:53 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
# at 123
#231125 14:13:53 server id 6  end_log_pos 154 CRC32 0x3fc1f191  Previous-GTIDs
# [empty]
# at 154
#231125 15:04:44 server id 6  end_log_pos 219 CRC32 0x676b3f18  Anonymous_GTID  last_committed=0        sequence_number=1       rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#231125 15:04:44 server id 6  end_log_pos 326 CRC32 0x8cc2c541  Query   thread_id=3     exec_time=0     error_code=0
SET TIMESTAMP=1700895884/*!*/;
SET @@session.pseudo_thread_id=3/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1436549152/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,@@session.collation_server=45/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
create database test charset=utf8
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```



**3.2.7 基于binlog数据恢复**

示例：创建一个test库，删除test库，需要恢复该库

恢复：

1. 查看当前正在使用的binlog文件

```mysql
/* 登入数据库中操作 */
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      326 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```



2. 查看对应的二进制文件内容

```mysql
mysql> show binlog events in 'mysql-bin.000001';
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| Log_name         | Pos | Event_type     | Server_id | End_log_pos | Info                                  |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
| mysql-bin.000001 |   4 | Format_desc    |         6 |         123 | Server ver: 5.7.43-log, Binlog ver: 4 |
| mysql-bin.000001 | 123 | Previous_gtids |         6 |         154 |                                       |
| mysql-bin.000001 | 154 | Anonymous_Gtid |         6 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
| mysql-bin.000001 | 219 | Query          |         6 |         326 | create database test charset=utf8     |
| mysql-bin.000001 | 326 | Anonymous_Gtid |         6 |         391 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'  |
| mysql-bin.000001 | 391 | Query          |         6 |         476 | drop database test                    |
+------------------+-----+----------------+-----------+-------------+---------------------------------------+
6 rows in set (0.00 sec)
```



3. 分析position号的起始位置

```shell
分析后得到如下position号：
起：154
末：326
```



4. 通过position号的起始位置截取二进制文件

```shell
# 登出数据库操作
mysqlbinlog --start-position=154 --stop-position=326 /data/mysql/binlog/mysql-bin.000001 > /tmp/bin.sql
```



5. 恢复数据

```mysql
/* 登入数据库中操作 */
set sql_log_bin=0;			# 临时关闭二进制日志，退出当前mysql会话时，二进制日志自动启动
source /tmp/bin.sql;		# 恢复数据
quit					   # 恢复完数据，要立即退出，避免后续的操作不会被记录在二进制日志里
```



## 3.3 GTID

**3.3.1 开启gtid**

```shell
[root@node01 ~]# vim /etc/my.cnf
[mysqld]
gtid-mode=on
enforce-gtid-consistency=true
[root@node01 ~]# systemctl restart mysqld
```

查看：如果未显示，可以创建一个数据库或插入一条数据后再查看

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+----------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                      |
+------------------+----------+--------------+------------------+----------------------------------------+
| mysql-bin.000004 |      316 |              |                  | 186c221c-8b50-11ee-878e-000c29688cba:1 |
+------------------+----------+--------------+------------------+----------------------------------------+
1 row in set (0.00 sec)

mysql> show variables like '%gtid%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)
```

**3.3.2 截取二进制日志**

默认使用GTID截取的二进制日志，不能恢复数据，因为GTID的幂等性（即执行过的语句不再执行）

```shell
# 截取日志
mysqlbinlog --include-gtids='186c221c-8b50-11ee-878e-000c29688cba:1-6' /data/mysql/binlog/mysql-bin.000004

# 截取日志并生成文件
mysqlbinlog --include-gtids='186c221c-8b50-11ee-878e-000c29688cba:1-6' /data/mysql/binlog/mysql-bin.000004 >/tmp/gtid.sql
```

正确的截取

- 正常截取

```shell
mysqlbinlog --skip-gtids --include-gtids='186c221c-8b50-11ee-878e-000c29688cba:1-6' /data/mysql/binlog/mysql-bin.000004 >/tmp/gtid.sql
```

- 跳过某些GTID

1--6，跳过3和4

```shell
mysqlbinlog --skip-gtids --include-gtids='186c221c-8b50-11ee-878e-000c29688cba:1-6' --
exclude-gtids='186c221c-8b50-11ee-878e-000c29688cba:3,186c221c-8b50-11ee-878e-000c29688cba:4'/data/mysql/binlog/mysql-bin.000004 >/tmp/gtid.sql
```

```shell
# 1--6，跳过3
mysqlbinlog --skip-gtids --include-gtids='186c221c-8b50-11ee-878e-000c29688cba:1-2:4-6' /data/mysql/binlog/mysql-bin.000004 >/tmp/gtid.sql
```





## 3.4 日志的其他操作

**3.4.1 临时关闭binlog**

```mysql
set sql_log_bin=0;		# 退出当前窗口，失效
```

**3.4.2 自动清理**

```mysql
# 查看自动清理周期
mysql> select @@expire_logs_days;
+--------------------+
| @@expire_logs_days |
+--------------------+
|                  0 |
+--------------------+
1 row in set (0.00 sec)

# 临时设置，重启失效
mysql> set global expire_logs_days=8;

# 永久设置
vim /etc/my.cnf
expire_logs_days=8

### 设置的周期
至少是一个全备周期+1，企业建议至少2个全备周期+1
```



# 4. 备份与恢复



## 4.1 备份方式及工具

### 4.1.1 逻辑备份

基于SQL语句进行备份

- mysqldump（MDP）
- mysqlbinlog

### 4.1.2 物理备份

基于磁盘数据文件备份

- xtrabackup(XBK) ：Percona 第三方  
- MySQL Enterprise Backup（MEB）



### 4.1.3 逻辑备份与物理备份比较

mysqldump

```shell
#优点：
1.不需要下载安装
2.备份出来的是SQL，文本格式，可读性高,便于备份处理
3.压缩比较高，节省备份的磁盘空间
#缺点：
4.依赖于数据库引擎，需要从磁盘把数据读出
然后转换成SQL进行转储，比较耗费资源，数据量大的话效率较低
#建议：
100G以内的数据量级，可以使用mysqldump
超过TB以上，我们也可能选择的是mysqldump，配合分布式的系统
1EB = 1024 PB = 1000000 TB
```

xtrabackup

```shell
#优点：
1.类似于直接cp数据文件，不需要管逻辑结构，相对来说性能较高
缺点：
2.可读性差
3.压缩比低，需要更多磁盘空间
#建议：
100G < xbk < TB
```



## 4.2 备份策略

### 4.2.1 备份方式

全备:全库备份，备份所有数据 

增量:备份变化的数据 逻

辑备份=mysqldump+mysqlbinlog 

物理备份=xtrabackup_full+xtrabackup_incr+binlog 或者 xtrabackup_full+binlog

### 4.2.2 备份周期

根据数据量设计备份周期 

比如：周日全备，周1-周6增量（数据量比较大） 

其他：通过主从复制备份 

看数据量集，如果数据量足够小的话，可以每天进行全备



## 4.3 mysqldump

### 4.3.1 基本应用

本地备份

```shell
mysqldump -uroot -pxxx -S /tmp/mysql.sock --set-gtid-purged=off
# 如果mysql.sock在tmp路径下那么该命令等价于：mysqldump -uroot -pxxx
```

远程备份

```shell
mysqldump -uroot -pxxx -hxxx -P3306 --set-gtid-purged=off
```

备份单个库或者多个库（test01和test02库）

```shell
mysqldump -uroot -p123 -B test01 test02 -S /tmp/mysql.sock --set-gtid-purged=off >/data/bak/db.sql
```

备份某个库下的单表或多表（test01下的t1与t2表）

```shell
mysqldump -uroot -p123 test01 t1 t2 -S /tmp/mysql.sock --set-gtid-purged=off >/data/backup/tab.sql
## 注意：只会备份建表和插入语句，所以，要恢复前，需要手动创建库，并且use该库后，再恢复
```

企业全备参数

```shell
mysqldump -uroot -p123 -A -R -E --triggers --master-data=2 --single-transaction --set-gtid-purged=off

# 参数说明
#必加参数（1）
-R 在备份时，同时备份存储过程和函数，如果没有会自动忽略
-E 在备份时，同时备份EVENT，如果没有会自动忽略
--triggers 在备份时，同时备份触发器，如果没有会自动忽略
#必加参数(2)
--master-data=2
#记录备份开始时 position号 ，可以作为将来做日志截取的起点。
1. 记录备份时的position
2. 自动锁表
3. 配合--single-transaction，减少锁的(innodb引擎)
#对于innodb的表，实现快照备份，不锁表
--single-transaction

# 不是必须要加的
#--set-gtid-purged=OFF （GTID模式独有的参数）
--set-gtid-purged=OFF （GTID模式独有的参数）
作用, 去除gtid所有信息,在日常备份恢复时可加
在做主从复制的时候不能加此参数！！！
#--max-allowed-packet（报got a packet bigger than max_allowed_packet bytes这
个错的时候需要调整这个参数的大小）
--max-allowed-packet=256M
# 默认是4MB 然后视具体情况添加该参数
```

全备+恢复步骤

```shell
#1. mysqldump -uroot -p -A -R -E --master-data=2 --single-transaction --triggers --set-gtid-purged=off >/data/backup/full.sql
#2. pkill mysqld
#3. rm -rf /data/mysql/data/*
#4. mysqld --initialize-insecure --user=mysql --basedir=/application/mysql/ --datadir=/data/mysql/data/
#5. 进入mysql恢复数据
#6. set sql_log_bin=0;
#7. source /data/backup/full.sql
#8. flush privileges;
```

### 4.3.2 企业案例（mysqldump+binlog）

```shell
#企业的备份恢复案例（mysqldump+binlog），年终故障恢复演练。4核 8G
#案例背景： 某中小型互联网公司。MySQL 5.7.28 ，Centos 7.6 ，数据量级70G，每日数据增量5-6M
#备份策略： 每天mysqldump全备+binlog备份，每天23:00进行。
#故障描述： 周三下午5点，数据由于某原因数据损坏。
#处理思路：
1. 挂出维护页
2. 评估一下数据损坏状态
1 全部丢失-->推荐直接生产恢复
	(1)测试库进行全备恢复
3. 恢复全备，将数据追溯到周二晚上23:00状态
4. 截取并恢复从备份时刻，到下午5点误删除之前binlog。
5. 校验数据一致性
6. 撤维护页，恢复生产。
7. 处理结果：
	经过30-40分钟处理，业务恢复
	评估此次故障的处理的合理性和实用性
```

案例模拟

```shell
# 全备
mysqldump -uroot -p123 -A -R --triggers -E --master-data=2 --set-gtid-purged=off --single-transaction >/data/backup/full.sql
# 模拟周三数据变化
create database mdb charset utf8mb4;
# 模拟数据损坏
rm -rf /data/mysql/data/*
pkill mysqld
rm -rf /data/mysql/data/*
# 进行数据初始化，启动数据库
mysqld --initialize-insecure --user=mysql --basedir=/usr/loca/mysql --datadir=/data/mysql/data
systemctl start mysqld
mysqladmin -hlocalhost -uroot password CloudEasy@2020
mysql -uroot -pCloudEasy@2020 -NBe "update mysql.user set host = '%' where user = 'root';"
mysql -uroot -pCloudEasy@2020 -NBe "flush privileges;"
# 进行全备恢复
mysql -uroot -pCloudEasy@2020
set sql_log_bin=0;
source /data/backup/full.sql
# 进行binlog恢复(先截取要恢复的信息)
mysql -uroot -pCloudEasy@2020
show master status;
show binlog events in 'mysql-bin.000001';
mysqlbinlog --start-position=2557 --stop-position=5053 /data/binlog/mysql-bin.000001 >/data/backup/b.sql
mysql -uroot -pCloudEasy@2020
set sql_log_bin=0;
source /data/backup/b.sql
# 验证数据
```



## 4.4 Percona-xtrabackup （XBK）

### 4.4.1 优点

1.直接拷贝物理文件，备份和恢复数据的速度非常快、安全可靠

2.在备份期间执行的事务不会间断，备份InnoDB数据不影响业务

3.备份期间不增加太多数据库的性能压力

4.支持对备份的数据自动校验

5.支持全量、增量、压缩备份及流备份

6.技持在线迁移表以及快速创建新的从库

7.支持几乎所有版本的MySQL和MariaDB



### 4.4.2 安装

```shell
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum -y install perl perl-devel libaio libaio-devel perl-Time-HiRes perl-DBD-MySQL libev
wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.12/binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.12-1.el7.x86_64.rpm
yum localinstall -y percona-xtrabackup-24-2.4.12-1.el7.x86_64.rpm
```

### 4.4.3 全备

```shell
innobackupex --user=root --password=123 -S /tmp/mysql.sock --no-timestamp /data/xbk/

# 参数说明
--user	用户名
--password	密码，空密码为 --password=''
-S	套接字，默认值为/tmp/mysql.sock，
--no-timestamp	取消时间戳，避免恢复因为时间问题出错

# 自定义备份路径名
innobackupex --user=root --password=123 -S /tmp/mysql.sock --no-timestamp /data/xbk/full_$(date +%F)
```



### 4.4.4 增量备份

基于上一次备份的增量

增量备份不能单独恢复，必须合并到全备中，一起恢复

```shell
innobackupex --user=root --password=123 -S /tmp/mysql.sock --no-timestamp --incremental --incremental-basedir=/data/bak/full_2023-11-29 /data/xbk/inc_$(date +%F)

# 参数说明
--no-timestamp	取消时间戳，避免恢复因为时间问题出错
--incremental	增量
--incremental-basedir	增量依赖哪个备份进行增倍
```



4.4.5 企业案例（XBK+XBK_inc）

```shell
#案例背景： 某中型互联网公司。MySQL 5.7.28 ，Centos 7.6 ，数据量级800G，每日数据增量20-
50M
#备份策略： 周日XBK全备 +周一到周六inc增量+binlog备份，每天23:00进行。
#故障描述： 周三下午2点，数据由于某原因数据损坏。
#处理思路：
1. 挂出维护页
2. 评估一下数据损坏状态
	1 全部丢失-->推荐直接生产恢复
	2 部分丢失
3. 整理合并所有备份：full+inc1+inc2
4. 截取 周二晚上到周三下午午故障点的binlog日志
5. 恢复全备，恢复binlog
6. 检查数据完整性
7. 恢复业务
8. 处理结果：
	1. 经过70-80分钟处理，业务恢复
	2. 评估此次故障的处理的合理性和实用性
```

模拟数据恢复

1. 删除数据

```shell
rm -rf /data/xbk/*
```

2. 模拟周日进行全备

```shell
innobackupex --user=root --password=CloudEasy@2020 -S /tmp/mysql.sock --no-timestamp /data/xbk/full_$(date +%F)
```

3. 模拟周一数据变化
4. 模拟周一晚上增量备份

```shell
innobackupex --user=root --password=CloudEasy@2020 -S /tmp/mysql.sock --no-timestamp --incremental --incremental-basedir=/data/xbk/full_2023-11-29 /data/xbk/inc_$(date +%F)
```

5. 模拟周二白天的数据变化
6. 模拟周二晚上增量备份

```shell
innobackupex --user=root --password=CloudEasy@2020 -S /tmp/mysql.sock --no-timestamp --incremental --incremental-basedir=/data/xbk/inc_2023-11-29 /data/xbk/inc2_$(date +%F)
```

7. 模拟删库

```shell
rm -rf /data/mysql/data/*
```

8. XBK增量恢复
   - 合并所有增量到全备（最好把周日全备再备份一下）

```shell
#1.处理原始全备
innobackupex --apply-log --redo-only /data/xbk/full_2023-11-29/

#2.整理并合并周一增量到全备
innobackupex --apply-log --redo-only --incremental-dir=/data/xbk/inc_2023-11-29/ /data/xbk/full_2023-11-29

#3.查看验证checkpoint（周日的群被的las_lsn号和周一增量的last_lsn号）
 cat /data/xbk/full_2023-11-29/xtrabackup_checkpoints /data/xbk/inc_2023-11-29/xtrabackup_checkpoints

#4.整理合并周二增量到全备（注意：最后一次增量不需要加redo-olny参数）
innobackupex --apply-log --redo-only --incremental-dir=/data/xbk/inc2_2023-11-29/ /data/xbk/full_2023-11-29/

#5.再次验证checkpoint
cat /data/xbk/full_2023-11-29/xtrabackup_checkpoints /data/xbk/inc2_2023-11-29/xtrabackup_checkpoints
```

- 再次整理全备

```shell
innobackupex --apply-log /data/xbk/full_2023-11-29/
```

- 清库

```shell
rm -rf /data/mysql/data/*
```

- 恢复全备

```shell
innobackupex --copy-back /data/bak/full_2019-11-29/
```

- 恢复数据

```shell
cp -a /data/xbk/full_2023-11-29/* /data/mysql/data/
chown -R mysql.mysql /data/mysql/data
systemctl restart mysqld
```

- 增量到故障时间点的数据需要基于binlog进行恢复（基于position号或GTID）

9. 验证数据

# 5. 主从复制

基于二进制日志复制的 

主库的修改操作会记录二进制日志 

从库会请求新的二进制日志并回放，最终达到主从数据同步

## 5.1 主从复制原理

![](/img/md/MySQL入门/1.png)

1. 名词认识

```shell
#文件：
	主库：master
	从库：
		relay-log 中继日志
		master.info 主库信息文件
		relay-log.info 中继日志应用信息
#线程：
	主库：
		binlog_dump_thread 二进制日志投递线程
		mysql -uroot -p -e "show processlist"
	从库:
		IO_Thread : 从库IO线程 ： 请求和接收binlog
		SQL_Thread: 从库的SQL线程 ： 回放日志
```

2. 工作原理

```shell
# 1.从库执行 change master to 语句，会立即将主库信息记录到master.info中
# 2.从库执行 start slave语句，会立即生成IO_T和SQL_T
# 3.IO_T 读取master.info文件，获取到主库信息
# 4.IO_T 连接主库，主库会立即分配一个DUMP_T,进行交互
# 5.IO_T 根据master.info binlog信息，向DUMP_T请求最新的binlog
# 6.主库DUMP_T,经过查询，如果发现有新的，截取并反回给从库IO_T
# 7.从库IO_T会收到binlog，存储在TCP/IP缓存中,在网络底层返回ACK
# 8.从库IO_T会更新master.info ,重置binlog位置点信息
# 9.从库IO_T会将binlog，写入到relay-log中
# 10.从库SQL_T 读取Relay-log.info 文件，获取上次执行过的位置点
# 11.SQL_T按照位置点往下执行relaylog日志
# 12.SQL_T执行完成后，重新更新relay-log.info
# 13.relaylog定期自动清理
#补充：
	#主库发生了信息的修改，更新binlog二进制日志完成后，会发送一个“信号”给Dump_T,Dump_T通知给IO_T线程来取新的，保证复制的实时性
```



## 5.2  基础主从复制（多节点）

1. 搭建主库：192.168.1.161

基础主从中只有主库开启binlog就可以

```cnf
[mysqld]
server_id=6
[mysql]
prompt=master [\\d]>
```

2. 搭建从库：192.168.1.162

```cnf
[mysqld]
server_id=7
[mysql]
prompt=slave01 [\\d]
```

3. 主库创建复制用户

```mysql
grant replication slave on *.* to repl@'%' identified by 'CloudEasy@2020';
```

4. 从库远程主库进行全备

```shell
mkdir -p /data/backup
mysqldump -uroot -pCloudEasy@2020 -h192.168.1.161 -P3306 -A -R -E --master-data=2 --triggers --single-transaction --set-gtid-purged=off >/data/backup/full.sql
```

5. 恢复数据到从库

```mysql
set sql_log_bin=0;
source /data/backup/full.sql
flush privileges;
exit
```

6. 从库：change master to

```mysql
# help change master to 可以获取模板。主库 "show master status;" 可获取 MASTER_LOG_FILE、MASTER_LOG_POS
CHANGE MASTER TO
  MASTER_HOST='192.168.1.161',
  MASTER_USER='repl',
  MASTER_PASSWORD='CloudEasy@2020',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='mysql-bin.000015',
  MASTER_LOG_POS=549,
  MASTER_CONNECT_RETRY=10;
```

7. 开启从库

```mysql
start slave;
```

8. 验证从库是否正常

```mysql
show slave status \G
# 关注是否均为 yes
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

9. 查看主库线程

```mysql
show processlist;
```

10. 从库信息配置错误导致从库状态异常，恢复步骤

```mysql
# 从库执行
stop slave;
reset slave all;
重新执行change master to
start slave;
```



## 5.3 主从故障检测

```shell
# 检测从库状态
mysql -uroot -p -e "show slave status \G"| grep "Running:"
# 检测从库报错
mysql -uroot -p -e "show slave status \G"| grep "Last"
```



## 5.4 主从复制高级进阶（基于GTID复制）

1. 主库搭建：192.168.1.161

```cnf
[mysqld]
server_id=6
# 启动GTID类型，否则就是普通的复制架构
gtid-mode=on
# 强制GTID的⼀致性
enforce-gtid-consistency=true
# slave更新是否记⼊⽇志
log-slave-updates=1
[mysql]
prompt=master [\\d]>
```

2. 从库搭建：192.168.1.162

```cnf
[mysqld]
server_id=7
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
[mysql]
prompt=slave01 [\\d]
```

3. 主库创建复制用户

```mysql
grant replication slave on *.* to repl@'%' identified by 'CloudEasy@2020';
```

4. 从库远程主库进行全备

```shell
mysqldump -uroot -pCloudEasy@2020 -h192.168.1.161 -P3306 -A -R -E --master-data=2 --triggers --single-transaction --set-gtid-purged=off >/tmp/full.sql
```

5. 恢复数据到从库

```mysql
set sql_log_bin=0;
source /tmp/full.sql
flush privileges;
exit
```

6. 从库配置 change master to

```mysql
CHANGE MASTER TO
  MASTER_HOST='192.168.1.161',
  MASTER_USER='repl',
  MASTER_PASSWORD='CloudEasy@2020',
  MASTER_AUTO_POSITION=1;				# 1 为自动获取
```

7. 从库启动

```mysql
start slave;
```

8. 检查主从状态

```mysql
# 从库
show slave status \G
# 主库
show processlist;
```



## 5.5 半同步复制

```shell
1. 主库执⾏新的事务,commit时,更新 show master status\G ,触发⼀个信号给
2. binlog dump 接收到主库的 show master status\G信息,通知从库⽇志更新了
3. 从库IO线程请求新的⼆进制⽇志事件
4. 主库会通过dump线程传送新的⽇志事件,给从库IO线程
5. 从库IO线程接收到binlog⽇志,当⽇志写⼊到磁盘上的relaylog⽂件时,给主库ACK_receiver线程
6. ACK_receiver线程触发⼀个事件,告诉主库commit可以成功了
7. 如果ACK达到了我们预设值的超时时间,半同步复制会切换为原始的异步复制
```

## 5.6 过滤复制



# 6. 基于MHA高可用

MHA高可用框架是一次性的，成功切换主库后，MAH会停止工作



## 6.1 MHA环境搭建

1. 1主2从GTID
   - 主：192.168.1.161
   - 从1：192.168.1.162
   - 从2：192.168.1.163
2. 配置个节点互信

```shell
# 主库操作
rm -rf /root/.ssh
ssh-keygen
cd /root/.ssh

scp -r /root/.ssh/ ssh root@192.168.1.162:/root
scp -r /root/.ssh/ ssh root@192.168.1.163:/root
```

3. 验证免密登陆

```shell
# 主
ssh 192.168.1.161 date
ssh 192.168.1.162 date
ssh 192.168.1.163 date
# 从1
ssh 192.168.1.161 date
ssh 192.168.1.162 date
ssh 192.168.1.163 date
#从2
ssh 192.168.1.161 date
ssh 192.168.1.162 date
ssh 192.168.1.163 date
```

4. 安装软件（关闭防火墙和selinux）

```shell
# 所有节点上传软件包：MHA.zip
unzip MHA.zip
# 所有节点安装依赖
yum install perl-DBD-MySQL -y
# 所有节点安装node
rpm -ivh /root/MHA/mha4mysql-node-0.56-0.el6.noarch.rpm
# 在主库创建mha用户
grant all on *.* to mha@'%' identified by 'CloudEasy@2020';
# Manager软件安装依赖（装在slave02节点机器上）
yum install -y perl-Config-Tiny epel-release perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes
# Manager软件安装（slave02节点执行）
rpm -ivh /root/MHA/mha4mysql-manager-0.56-0.el6.noarch.rpm
```

4. MHA Manager 配置文件准备(slave02节点)

```shell
mkdir -p /etc/mha && mkdir -p /var/log/mha/app1

vim /etc/mha/app1.cnf

[server default]
manager_log=/var/log/mha/app1/manager
manager_workdir=/var/log/mha/app1
master_binlog_dir=/data/mysql/binlog
user=mha
password=CloudEasy@2020
ping_interval=2
repl_password=CloudEasy@2020
repl_user=repl
ssh_user=root
[server1]
hostname=192.168.1.161
port=3306
[server2]
hostname=192.168.1.162
port=3306
[server3]
hostname=192.168.1.163
port=3306



### 参数说明
#！！！！repl_user 写你主从授权的用户名
#！！！！repl_password 写你主从授权的用户的密码
#！！！！server1 hostname 主库ip
#！！！！server2 hostname slave01 ip
#！！！！server3 hostname slave02 ip
#！！！！master_binlog_dir 主库二进制日志的路径


#1.vim /etc/mha/app1.cnf app1名字不固定，因为MHA可以管理多集群，可以是app1，app2，用名称来区分具体的业务，如app2下又对应三个不同节点信息，就代表另一个业务的一主双从
#2.vim /etc/mha/app1.cnf 主库二进制文件路径，用于补偿数据时候来使用
#3.ping_interval 时间间隔 ping->pong 2=2s 用于检测主库状态能ping通表示主库可用，ping不通表示主库down了
#4.repl_password=123456 repl_user=repl 主从复制授权用户名和密码，当发现主库down了之后，MHA会自动切换主从，所以要知道用户名密码
```

5. 检查状态(slave02节点操作)

```shell
# 1. 互信检查（如果失败，需要重新执行步骤2，步骤3）
masterha_check_ssh --conf=/etc/mha/app1.cnf

# 2. 主从状态检查（如果失败，重新搭建GTID主从）
masterha_check_repl --conf=/etc/mha/app1.cnf
```

6. 开启MHA(slave02节点操作)

```shell
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null> /var/log/mha/app1/manager.log 2>&1 &

### 参数解读 以下内容不用输入 ####
#1.--remove_dead_master_conf 自动移除故障点，从配置文件里（/etc/mha/app1.cnf）
#2.--ignore_last_failover MHA不能短时间内多次切换，如加次参数便可
```

7. 检查状态

```shell
masterha_check_status --conf=/etc/mha/app1.cnf
```



## 6.2 MHA恢复

1. 模拟停止主库

```
systemctl stop mysqld
```

2. 查看当前主库及MHA配置文件

```shell
# 登陆原从库，查看当前主库
show slave status\G
show processlist;

# 查看当前MHA配置文件是否缺失server1配置
vim /etc/mha/app1.cnf
```

3. 修复MHA

```shell
# 1. 修复原主库
systemctl start mysqld

# 2. 修复主从关系：将原主库以从库身份加入主从架构中
CHANGE MASTER TO
  MASTER_HOST='192.168.1.162',
  MASTER_USER='repl',
  MASTER_PASSWORD='CloudEasy@2020',
  MASTER_AUTO_POSITION=1;

start slave;
show slave status\G

# 3. 修复MHA配置文件
vim /etc/mha/app1.cnf
##手动添加被删除的server1配置
[server1]
hostname=192.168.1.161
port=3306

# 4. 启动MHA
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null> /var/log/mha/app1/manager.log 2>&1 &

# 5. 检查状态
masterha_check_status --conf=/etc/mha/app1.cnf

# 6. 切换为初始主从架构（可选）
将当前主库停掉，MHA会自动将主切换为server1（即原主库）
手动将后主以从库身份加入主从架构，进行MHA修复即可
```



## 6.3 邮件提醒

- slave01节点操作

```shell
# 安装依赖
yum install perl -y

# 添加配置
vim /etc/mha/app1.cnf
report_script=/usr/local/bin/send

# 将email目录下的所有文件拷贝至/usr/local/bin/目录下
cd /root/MHA && unzip email.zip && chmod +x email/ && cp email/* /usr/local/bin/

# 修改文件
vim /usr/local/bin/testpl		# 修改发送端邮箱、接收邮箱、发送端邮箱的用户名以及客户端独立授权码（非登陆的密码）、发送端邮箱的协议（如 smtp.163.com）

# 服务器开通安全组（虚拟机不需要开通）：25端口
# 测试发送
./testpl

# 重启MHA
masterha_stop --conf=/etc/mha/app1.cnf
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &

# 模拟故障：停止主库
systemctl stop mysqld

# 观察是否接收到邮件
# 修复主从及MHA
```



## 6.4 MHA VIP

要求：主从服务器均在同一网段

效果：VIP随主库切换而漂移（主库用来写操作，即代码不需要跟随主库切换而修改连接地址）

```shell
#1.移动文件
cp master_ip_failover.txt /usr/local/bin/master_ip_failover

#2.
cd /usr/local/bin/
yum install -y dos2unix
dos2unix master_ip_failover
chmod +x master_ip_failover

#3.修改配置文件
vim /etc/mha/app1.cnf
master_ip_failover_script=/usr/local/bin/master_ip_failover

#4.修改文件内容
vim /usr/local/master_ip_failover
	##参数说明
		# VIP
		my $vip = '192.168.0.100/24';
		my $key = '1';
		# eth0对应你自己的网卡名
		my $ssh_start_vip = "/sbin/ifconfig eth0:$key $vip";
		my $ssh_stop_vip = "/sbin/ifconfig eth0:$key down";
		# 主库上，手工生成第一个vip地址
		ifconfig eth0:1 192.168.0.100/24

#5.重启MHA
masterha_stop --conf=/etc/mha/app1.cnf
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
```



## 6.5 读写分离--Atlas(360)



1. 架构

- 主：192.168.1.161	server_id=6
- 从1：192.168.1.162  server_id=7
- 从2：192.168.1.163  server_id=8



2. 效果

- 主库：只写
- 从库：只读，且自带负载均衡



3. 搭建--slave02节点操作

```shell
#1.安装工具
rpm -ivh Atlas-2.2.1.el6.x86_64.rpm

#2.编辑配置文件
cd /usr/local/mysql-proxy/conf
mv test.cnf test.cnf.bak

cat > test.cnf <<EOF
[mysql-proxy]
admin-username = user
admin-password = pwd
proxy-backend-addresses = 152.136.108.121
proxy-read-only-backend-addresses = 121.40.114.23:3306,212.64.56.172:3306
pwds = repl:3yb5jEku5h4=,mha:O2jBXONX098=
daemon = true
keepalive = true
event-threads = 8
log-level = message
log-path = /usr/local/mysql-proxy/log
sql-log=ON
proxy-address = 0.0.0.0:33060
admin-address = 0.0.0.0:2345
charset=utf8
EOF

	##配置参数解读
	# 1.proxy-backend-addresses 写入节点IP （如果之前设置了VIP此处可以直接写VIP地址）
	# 2.proxy-read-only-backend-addresses 读取数据节点IP
	# 3.密码需要加密 cd /usr/local/mysql-proxy/bin/ && ./encrypt 123456 ，会返回加密后的密码
	
#3.启动（test是配置文件名称）
/usr/local/mysql-proxy/bin/mysql-proxyd test start

#4.查看监听端口
netstat -lntup|grep proxy
```



4. 验证读

```shell
#1.登陆mha账号 端口33060
mysql -umha -p -hIP地址写你安装altas这台机器的IP地址 -P33060
#2.进行查询
select @@server_id;
```



5. 验证写

```shell
#1.登陆mha账号 端口33060
mysql -umha -p -hIP地址写你安装altas这台机器的IP地址 -P33060

#2.进行验证
begin;select @@server_id; commit;
```



6. 用户管理

```shell
#1.生产授权用户
主库节点操作
grant all on *.* to root@'%' identified by '123456';
#2.密码加密
cd /usr/local/mysql-proxy/bin/
./encrypt 123456
/iZxz+0GRoA=
#3.修改配置文件
vim /usr/local/mysql-proxy/conf/test.cnf
在pwds=后边继续添加
pwds = repl:/iZxz+0GRoA=,mha:O2jBXONX098=,root:/iZxz+0GRoA=
#4.重启
/usr/local/mysql-proxy/bin/mysql-proxyd test restart
#5.登陆测试
mysql -uroot -p123 -h106.12.182.209 -P33060
```



9. 常用命令

```mysql
# 查看节点信息
select * from backend;

# 删除节点
remove backend 3;

# 添加节点
add slave 192.168.1.164:33036

# 下线节点
SET OFFLINE $backend_id ;

# 上线节点
SET ONLINE $backend_id;

# 查看用户
SELECT * FROM pwds;

# 删除用户
REMOVE PWD $pwd;

# 添加用户--明文
ADD PWD root:123;

# 添加用户--密文
ADD ENPWD $pwd;
```



# 7. 基于Mycat分布式架构

## 7.1 架构

- 节点1（4G内存）：192.168.1.161	端口：3307	3308	3309	3310
- 节点2（4G内存）：192.168.1.162	端口：3307	3308	3309	3310

```shell
192.168.1.161:3307 与 192.168.1.162:3307	互为主从，其中 192.168.1.162:3307 是备主，即当前162:3307是从，161:3307故障后，162:3307才切换为主（此时161:3307是备主）
192.168.1.161:3307 是 192.168.1.161:3309 的主
192.168.1.162:3307 是 192.168.1.162:3309 的主
#161:3307作为主库时，161:3309+162:3307+162:3309均为从库
#161:3307故障后，162:3307切换为主库，162:3309为从库，此时161:3309不参与主从架构
#161:3307故障修复后，162:3307切换为主库，162:3309+161:3307+161:3309均为从库

192.168.1.161:3308 与 192.168.1.162:3308 互为主从，其中 192.168.1.162:3308 是备主，即当前162:3308是从，当161:3308故障后，162:3308才切换为主（此时161:3308是备主）
192.168.1.161:3308 是 192.168.1.161:3310 的主
192.168.1.162:3308 是 192.168.1.162:3310 的主
```



## 7.2 基础环境搭建

1. 基础环境：2节点均执行

```shell
#1.删除默认MariaDB并添加mysql用户
yum remove `rpm -qa| grep mariadb` -y
useradd -s /sbin/nologin mysql
yum install libaio -y

#2.下载软件包并解压
# yum install wget -y
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz
tar -xf mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
ln -sv /usr/local/mysql-5.7.43-linux-glibc2.12-x86_64 /usr/local/mysql

#3.创建初始化数据目录
mkdir -p /data/33{07..10}/data
chown -R mysql.mysql /data/*
mysqld --initialize-insecure --user=mysql --datadir=/data/3307/data --basedir=/usr/loca/mysql
mysqld --initialize-insecure --user=mysql --datadir=/data/3308/data --basedir=/usr/loca/mysql
mysqld --initialize-insecure --user=mysql --datadir=/data/3309/data --basedir=/usr/loca/mysql
mysqld --initialize-insecure --user=mysql --datadir=/data/3310/data --basedir=/usr/loca/mysql
```

2. 写入配置文件：节点1执行

```shell
cat >/data/3307/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3307/data
socket=/data/3307/mysql.sock
port=3307
log-error=/data/3307/mysql.log
log_bin=/data/3307/mysql-bin
binlog_format=row
skip-name-resolve
server-id=7
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3308/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3308/data
port=3308
socket=/data/3308/mysql.sock
log-error=/data/3308/mysql.log
log_bin=/data/3308/mysql-bin
binlog_format=row
skip-name-resolve
server-id=8
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3309/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3309/data
socket=/data/3309/mysql.sock
port=3309
log-error=/data/3309/mysql.log
log_bin=/data/3309/mysql-bin
binlog_format=row
skip-name-resolve
server-id=9
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3310/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3310/data
socket=/data/3310/mysql.sock
port=3310
log-error=/data/3310/mysql.log
log_bin=/data/3310/mysql-bin
binlog_format=row
skip-name-resolve
server-id=10
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/etc/systemd/system/mysqld3307.service<<EOF
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.targe
After=syslog.target
[Install]
WantedBy=multi-user.target
[Service]
User=mysql
Group=mysql
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3307/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3308.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3308/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3309.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3309/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3310.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3310/my.cnf
LimitNOFILE = 5000
EOF
```

3. 写入配置文件：节点2执行

```shell
cat >/data/3307/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3307/data
socket=/data/3307/mysql.sock
port=3307
log-error=/data/3307/mysql.log
log_bin=/data/3307/mysql-bin
binlog_format=row
skip-name-resolve
server-id=17
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3308/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3308/data
port=3308
socket=/data/3308/mysql.sock
log-error=/data/3308/mysql.log
log_bin=/data/3308/mysql-bin
binlog_format=row
skip-name-resolve
server-id=18
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3309/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3309/data
socket=/data/3309/mysql.sock
port=3309
log-error=/data/3309/mysql.log
log_bin=/data/3309/mysql-bin
binlog_format=row
skip-name-resolve
server-id=19
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/data/3310/my.cnf<<EOF
[mysqld]
basedir=/usr/loca/mysql
datadir=/data/3310/data
socket=/data/3310/mysql.sock
port=3310
log-error=/data/3310/mysql.log
log_bin=/data/3310/mysql-bin
binlog_format=row
skip-name-resolve
server-id=20
gtid-mode=on
enforce-gtid-consistency=true
log-slave-updates=1
EOF

cat >/etc/systemd/system/mysqld3307.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3307/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3308.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3308/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3309.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3309/my.cnf
LimitNOFILE = 5000
EOF

cat >/etc/systemd/system/mysqld3310.service<<EOF
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
ExecStart=/usr/loca/mysql/bin/mysqld --defaults-file=/data/3310/my.cnf
LimitNOFILE = 5000
EOF
```

4. 修改权限，启动多实例：2节点均执行

```shell
#授权
chown -R mysql.mysql /data/*
#启动实例
systemctl start mysqld3307
systemctl start mysqld3308
systemctl start mysqld3309
systemctl start mysqld3310
#验证：查看server_id
mysql -S /data/3307/mysql.sock -e "show variables like 'server_id'"
mysql -S /data/3308/mysql.sock -e "show variables like 'server_id'"
mysql -S /data/3309/mysql.sock -e "show variables like 'server_id'"
mysql -S /data/3310/mysql.sock -e "show variables like 'server_id'"
```

## 7.3 配置主从复制

以下箭头指向主库

- shard1

```shell
# 1. 192.168.1.161:3307 <---> 192.168.1.162:3307
##节点2：192.168.1.162
mysql -S /data/3307/mysql.sock -e "grant replication slave on *.* to repl@'%' identified by '123';"
mysql -S /data/3307/mysql.sock -e "grant all on *.* to root@'%' identified by '123' with grant option;"
##节点1：192.168.1.161
mysql -S /data/3307/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.162', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3307/mysql.sock -e "start slave;"
mysql -S /data/3307/mysql.sock -e "show slave status\G"|grep Running:
##节点2：192.168.1.162
mysql -S /data/3307/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.161', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3307/mysql.sock -e "start slave;"
mysql -S /data/3307/mysql.sock -e "show slave status\G"|grep Running:

# 2. 192.168.1.161:3309 ---> 192.168.1.161:3307
##节点1：192.168.1.161
mysql -S /data/3309/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.161', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3309/mysql.sock -e "start slave;"
mysql -S /data/3309/mysql.sock -e "show slave status\G"|grep Running:

#3. 192.168.1.161:3309 --> 192.168.1.162:3307
##节点2
mysql -S /data/3309/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.162', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3309/mysql.sock -e "start slave;"
mysql -S /data/3309/mysql.sock -e "show slave status\G"|grep Running:
```

- shard2

```shell
#1. 192.168.1.161:3308 <---> 192.168.1.162:3308
##节点1：192.168.1.161
mysql -S /data/3308/mysql.sock -e "grant replication slave on *.* to repl@'%' identified by '123';"
mysql -S /data/3308/mysql.sock -e "grant all on *.* to root@'%' identified by '123' with grant option;"
##节点2：192.168.1.162
mysql -S /data/3308/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.161', MASTER_PORT=3308, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3308/mysql.sock -e "start slave;"
mysql -S /data/3308/mysql.sock -e "show slave status\G"|grep Running:
##节点1：192.168.1.161
mysql -S /data/3308/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.162', MASTER_PORT=3308, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3308/mysql.sock -e "start slave;"
mysql -S /data/3308/mysql.sock -e "show slave status\G"|grep Running:

#2. 192.168.1.162:3310 --> 192.168.1.162:3308
##节点2：192.168.1.162
mysql -S /data/3310/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.162', MASTER_PORT=3308, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3310/mysql.sock -e "start slave;"
mysql -S /data/3310/mysql.sock -e "show slave status\G"|grep Running:

#3. 192.168.1.161:3310 --> 192.168.1.161:3308
##节点1：192.168.1.161
mysql -S /data/3310/mysql.sock -e "CHANGE MASTER TO MASTER_HOST='192.168.1.161', MASTER_PORT=3308, MASTER_AUTO_POSITION=1, MASTER_USER='repl', MASTER_PASSWORD='123';"
mysql -S /data/3310/mysql.sock -e "start slave;"
mysql -S /data/3310/mysql.sock -e "show slave status\G"|grep Running:
```

- 检查主从状态（每个节点8个yes代表正确）：2节点均操作

```shell
mysql -S /data/3307/mysql.sock -e "show slave status\G"|grep Running:
mysql -S /data/3308/mysql.sock -e "show slave status\G"|grep Running:
mysql -S /data/3309/mysql.sock -e "show slave status\G"|grep Running:
mysql -S /data/3310/mysql.sock -e "show slave status\G"|grep Running:
```



## 7.4 Mycat安装

任意一台节点操作均可

```shell
#1.安装java环境
yum install java -y

#2.下载mycat
wget http://dl.mycat.io/1.6.7.3/20190927161129/Mycat-server-1.6.7.3-release-20190927161129-linux.tar.gz
tar -xf Mycat-server-1.6.7.3-release-20190927161129-linux.tar.gz -C /usr/local/
ln -sv /usr/local/xxxx /usr/local/mycat
echo -e 'export MYCAT_HOME=/usr/local/mycat\nexport PATH=$PATH:$MYCAT_HOME/bin' >>  /etc/profile.d/mycat.sh
source /etc/profile.d/mycat.sh

#3.启动
mycat start

#4.连接mycat
###此处的root用户和密码与mysql中的root用户无关，是mycat本身的默认用户。在配置文件定义的：conf/server.
mysql -uroot -p123456 -h121.40.114.23 -P8066
```



## 7.5 Mycat数据配置---用户授权

```shell
#节点1：192.168.1.161
mysql -S /data/3307/mysql.sock
grant all on *.* to mycat@'%' identified by '123';
mysql -S /data/3308/mysql.sock
grant all on *.* to mycat@'%' identified by '123';
```



## 7.6 Mycat配置文件---读写分离

- 仅实现节点1（192.168.1.161）中3307写，3309读

1. 修改配置文件

```xml
cd /usr/local/mycat/
mv schema.xml schema.xml.bak
vim schema.xml

<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="sh1">
</schema>
	<dataNode name="sh1" dataHost="mycat" database= "mysql" />
	<dataHost name="mycat" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1">
		<heartbeat>select user()</heartbeat>
	<writeHost host="db1" url="192.168.1.161:3307" user="mycat" password="123">
			<readHost host="db2" url="192.168.1.161:3309" user="mycat" password="123" />
	</writeHost>
	</dataHost>
</mycat:schema>
```

```shell
######配置文件解读
#balance属性
负载均衡类型，目前的取值有3种：
1. balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。
2. balance="1"，全部的readHost与standby writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。
3. balance="2"，所有读操作都随机的在writeHost、readhost上分发。

#writeType属性
负载均衡类型，目前的取值有2种：
1. writeType="0", 所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后已切换后的为主，切换记录在配置文件中:dnindex.properties .
2. writeType=“1”，所有写操作都随机的发送到配置的writeHost，但不推荐使用

#switchType属性
-1 表示不自动切换
1 默认值，自动切换
2 基于MySQL主从同步的状态决定是否切换 ，心跳语句为 show slave status
```



1. 重启mycat

```shell
mycat restart
```

3. 测试读写分离

```shell
mysql -uroot -p123456 -h192.168.1.161 -P8066		# -h 指定安装mycat的节点IP
select @@server_id;	#验证读
begin;select @@server_id;commit;	#验证写
```



## 7.7 Mycat配置文件---高可用+读写分离

1. 修改配置文件

```xml
cd /usr/local/mycat/
vim schema.xml

<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="sh1">
</schema>
	<dataNode name="sh1" dataHost="mycat" database= "mysql" />
	<dataHost name="mycat" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1">
		<heartbeat>select user()</heartbeat>
	<writeHost host="db1" url="192.168.1.161:3307" user="mycat" password="123">
			<readHost host="db2" url="192.168.1.161:3309" user="mycat" password="123" />
	</writeHost>
	<writeHost host="db3" url="192.168.1.162:3307" user="mycat" password="123">
			<readHost host="db4" url="192.168.1.169:3309" user="mycat" password="123" />
	</writeHost>
	</dataHost>
</mycat:schema>
```

2. 重启mycat

```shell
mycat restart
```

3. 测试读写分离

```shell
mysql -uroot -p123456 -h192.168.1.161 -P8066		# -h 指定安装mycat的节点IP
select @@server_id;	#验证读
begin;select @@server_id;commit;	#验证写
```

4. 测试高可用

```shell
# 节点1挂掉机器 3307 
systemctl stop mysqld3307
mysql -uroot -p123456 -h192.168.1.161 -P8066
select @@server_id;	#验证读
begin;select @@server_id;commit;	#验证写
```

5. 修复架构

```shell
# 将节点1机器3307实例以节点2机器3307实例的从（备主）加入架构中即可
systemctl start mysqld3307		# 节点1操作

#验证
select @@server_id;	#验证读
begin;select @@server_id;commit;	#验证写
```

