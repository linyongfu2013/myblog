---
title: Mac 下使用 mysqld_multi 启动多个MySQL实例
tags: 
- MySQL
- Mac
categories: 
- MySQL
---

#### 环境介绍
* Mac 系统为最新的 macOS High Sierra
* MySQL 是使用 Homebrew 安装的，版本是`mysql: stable 5.7.21 (bottled), devel 8.0.3-rc`.
* 教程参考官网: https://dev.mysql.com/doc/refman/5.7/en/mysqld-multi.html

<!--more-->

#### 准备工作
本次示例使用`mysqld_multi` 安装6个 MySQL 实例，端口分别为 `3307 ~ 3312`。
1. 创建各实例的单独数据目录，我的目录在 `/usr/local/var`，data 目录是存放数据文件的，log 目录是存放慢查询日志文件的，执行如下命令:
```
mkdir -p mysql_3307/data mysql_3307/log mysql_3308/data mysql_3308/log mysql_3309/data mysql_3309/log mysql_3310/data mysql_3310/log mysql_3311/data mysql_3311/log mysql_3312/data mysql_3312/log
```

2. 同理，创建bin-log 日志文件目录，我的目录在 `/usr/local/var/log`：
```
mkdir -p mysql_3307 mysql_3308 mysql_3309 mysql_3310 mysql_3311 mysql_3312
```
3. 准备配置文件 `/usr/local/etc/my_multi.cnf`：
```
# Default Homebrew MySQL server config

[mysqld_multi]
mysqld     = /usr/local/Cellar/mysql/5.7.20/bin/mysqld_safe
mysqladmin = /usr/local/Cellar/mysql/5.7.20/bin/mysqladmin
user       = root
pass       = 123456  //注意不要写成password

[mysqld3307]
server-id  = 3307
port       = 3307
socket     = /tmp/mysql_3307.sock
pid-file   = /usr/local/var/mysql_3307/mysql.pid
datadir    = /usr/local/var/mysql_3307/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3307/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3307/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3307/general.log

[mysqld3308]
server-id  = 3308
port       = 3308
socket     = /tmp/mysql_3308.sock
pid-file   = /usr/local/var/mysql_3308/mysql.pid
datadir    = /usr/local/var/mysql_3308/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3308/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3308/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3308/general.log

[mysqld3309]
server-id  = 3309
port       = 3309
socket     = /tmp/mysql_3309.sock
pid-file   = /usr/local/var/mysql_3309/mysql.pid
datadir    = /usr/local/var/mysql_3309/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3309/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3309/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3309/general.log

[mysqld3310]
server-id  = 3310
port       = 3310
socket     = /tmp/mysql_3310.sock
pid-file   = /usr/local/var/mysql_3310/mysql.pid
datadir    = /usr/local/var/mysql_3310/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3310/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3310/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3310/general.log

[mysqld3311]
server-id  = 3311
port       = 3311
socket     = /tmp/mysql_3311.sock
pid-file   = /usr/local/var/mysql_3311/mysql.pid
datadir    = /usr/local/var/mysql_3311/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3311/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3311/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3311/general.log

[mysqld3312]
server-id  = 3312
port       = 3312
socket     = /tmp/mysql_3312.sock
pid-file   = /usr/local/var/mysql_3312/mysql.pid
datadir    = /usr/local/var/mysql_3312/data
language   = /usr/local/Cellar/mysql/5.7.20/share/mysql/english
user       = root
log-bin    = /usr/local/var/log/mysql_3312/mysql-bin
binlog_format = mixed
slow_query_log = on
slow_query_log_file = /usr/local/var/log/mysql_3312/slow.log
long_query_time = 1
log-queries-not-using-indexes
log_output = FILE,TABLE
general_log = on
general_log_file = /usr/local/var/log/mysql_3312/general.log

[mysqld]
max_connections = 2000
wait_timeout = 10000

validate_password = off

character_set_server=utf8
init_connect='SET NAMES utf8'

#skip-grant-tables
# Only allow connections from localhost
#bind-address = 127.0.0.1

```

#### 启动
执行以下命令启动所有实例：
```
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf start
```
此时留意当前命令行日志输出，每个实例会生成一个随机的初始密码，后面第一次登录时需要用到。

可以单独启动某个实例或部分实例，只要在命令后面指定具体的实例id即可，实例id为上面配置文件里标签`[mysqld3312]` 后面的数字:
```
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf start 3307
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf start 3307,3310-3312
```

同理，停止所有实例：
```
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf stop
```

停止指定实例：
```
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf stop 3307,3310-3312
```

可以使用 `report` 命令查看实例运行状态：
```
mysqld_multi --defaults-file=/usr/local/etc/my_multi.cnf report

Reporting MySQL servers
MySQL server from group: mysqld3307 is not running
MySQL server from group: mysqld3308 is running
MySQL server from group: mysqld3309 is running
MySQL server from group: mysqld3310 is running
MySQL server from group: mysqld3311 is running
MySQL server from group: mysqld3312 is running
```

#### 登录各实例服务器并修改密码
登录数据库需要指定各实例使用的socket文件，具体文件如上面配置文件所示。使用如下命令登录：
```
mysql -uroot -S/tmp/mysql_3307.sock -p
```
输入初始密码即可登录。登录后需要重置密码：
```
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```
授权root用户远程登录:
```
grant all on *.* to 'root'@'%' identified by '123456' with grant option;
flush privileges;
```
