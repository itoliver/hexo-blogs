---
title: docker-mariadb集群-主从同步
date: 2018-04-09 12:27:34
description:
categories: dockers
tags: [docker, mariadb, linux自动化运维]
copyright: true
---
前言
MariaDB数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可 MariaDB的目的是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。在存储引擎方面，使用XtraDB（英语：XtraDB）来代替MySQL的InnoDB。
<!--more-->
# 修改mariadb docker系统时间
```
tzselect
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
## master部署
# 1.创建mariadb数据配置文件
```
mkdir -p /opt/docker-mariadb/{data,conf.d,script}
```
# 2.拉取mairadb10.1.10镜像
```
docker pull mariadb:10.1.10
```
# 3.创建mariadb配置文件
```
tee /opt/docker-mariadb/conf.d/my.cnf <<EOF
[mysqld]
datadir=/var/lib/mysql
socket=/run/mysqld/mysqld.sock
server-id = 1
log_bin=master-bin
relay-log=relay-bin
expire_logs_days = 30
binlog_format=row
sync_binlog = 1
#sync_master_info = 1
default-storage-engine=INNODB
innodb_file_per_table = ON
#innodb_flush_logs_at_trx_commit
#innodb_support_xa = on
character-set-server=utf8mb4
slow_query_log = 1
long_query_time = 2
log_output = 'TABLE'
[mysql]
default-character-set=utf8mb4
auto-rehash

[mysqld_safe]
log-error=/var/log/mysql.log
pid-file=/run/mysqld/mysqld.pid

EOF
```
# 4.创建运行脚本
```
tee /opt/docker-mariadb/script/mariadb.sh<<EOF
#!/bin/bash
docker run -tid \
--restart=always \
--name mariadb-master \
-p 3307:3306 \
-v /opt/docker-mariadb/data:/var/lib/mysql \
-v /opt/docker-mariadb/conf.d:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=123456 \
mariadb:10.1.10
```
查看docker容器运行状态
```
docker ps -a
```
# 5.进入mariadb容器
```
docker exec -it mariadb-master bash
```
# 6.配置mysql master用户授权
```
mysql -uroot -p
grant replication slave,replication client on *.* to 'repluser'@'shangserver004' identified by 'replpassword';
flush privileges;
show master status;
```
## slave部署
# 1.创建mariadb数据配置文件
```
mkdir -p /opt/docker-mariadb/{data,conf.d,script}
```
# 2.拉取mairadb10.1.10镜像
```
docker pull mariadb:10.1.10
```
# 3.创建mariadb配置文件
```
tee /opt/docker-mariadb/conf.d/my.cnf <<EOF
[mysqld]
datadir=/var/lib/mysql
socket=/run/mysqld/mysqld.sock
server-id = 2
#log_bin=slave-bin
#binlog_format=row
relay-log=relay-bin
expire_logs_days = 30
read-only = on
sync_binlog = 1
default-storage-engine=INNODB
#sync_relay_log = 1
#sync_relay_log_info = 1
innodb_file_per_table = ON
character-set-server=utf8mb4
slow_query_log = 1
long_query_time = 2
log_output = 'TABLE'
[mysql]
default-character-set=utf8mb4
auto-rehash

[mysqld_safe]
log-error=/var/log/mysql.log
pid-file=/run/mysqld/mysqld.pid
EOF
```
# 4.创建运行脚本
```
tee /opt/docker-mariadb/script/mariadb.sh<<EOF
#!/bin/bash
docker run -tid \
--restart=always \
--name mariadb-slave \
-p 3307:3306 \
-v /opt/docker-mariadb/data:/var/lib/mysql \
-v /opt/docker-mariadb/conf.d:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=123456 \
mariadb:10.1.10
EOF
```
查看docker容器运行状态
```
docker ps -a
```
# 5.进入mariadb容器
```
docker exec -it mariadb-slave bash
```
# 6.配置mysql slave
```
mysql -uroot -p
```
```
change master to
master_host='192.168.1.14',
master_port=3307,
master_user='repluser',
master_password='replpassword',
master_log_file='master-bin.000001',
master_log_pos=674,
master_connect_retry=5,
master_heartbeat_period=2;
```
```
star slave;
```
```
show status slave;
```