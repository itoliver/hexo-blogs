---
title: docker-redis集群
date: 2018-04-08 11:16:46
categories: dockers
tags: [docker, redis, linux自动化运维]
---
前言
Redis是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。从2010年3月15日起，Redis的开发工作由VMware主持。从2013年5月开始，Redis的开发由Pivotal赞助。
此文只要是针对基于docker部署redis集群，实现主从同步
<!--more-->
master	192.168.1.14
slave	192.168.1.15

## master操作

# 1.创建数据文件
```
mkdir /opt/redis
```
# 2.拉取redis镜像
```
docker pull benyoo/redis:3.2.5
```
# 3.配置redis文件
```
echo 'bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""
databases 8
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
 
masterauth ZDU0NTlkNDY5NWZi
requirepass ZDU0NTlkNDY5NWZi' >/opt/redis/redis.conf
```
# 4.配置防火墙
```
iptables -I INPUT 5 -p tcp -m state --state NEW -m tcp -m comment --comment "REDIS_SERVER" -m multiport --dports 6379 -j ACCEPT
iptables -nvxL --lin
```
# 5.启动redis容器
```
docker run -d \
--privileged=true \
--name redis-master \
--restart=always \
-p 6379:6379
-v /opt/redis/redis.conf:/etc/redis.conf \
-v /etc/localtime:/etc/localtime \
benyoo/redis:3.2.5
```
## slave上操作

# 1.创建数据文件
```
mkdir /opt/redis
```
2.拉取redis镜像
```
docker pull benyoo/redis:3.2.5
```
3.配置redis文件
```
echo 'bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile ""
databases 8
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
 
slaveof 192.168.1.14 6379
masterauth ZDU0NTlkNDY5NWZi
requirepass ZDU0NTlkNDY5NWZi' >/opt/redis/redis.conf
```
# 4.配置防火墙
```
iptables -I INPUT 5 -p tcp -m state --state NEW -m tcp -m comment --comment "REDIS_SERVER" -m multiport --dports 6379 -j ACCEPT
iptables -nvxL --lin
```
# 5.启动redis容器
```
docker run -d \
--privileged=true \
--name redis-slave \
--restart=always \
-p 6379:6379
-v /opt/redis/redis.conf:/opt/redis/redis.conf \
-v /etc/localtime:/etc/localtime \
benyoo/redis:3.2.5
```
# 测试
```
docker exec -it redis redis-cli -h 192.168.1.15 -a ZDU0NTlkNDY5NWZi info replication
# Replication
role:slave
master_host:192.168.1.14
master_port:6379
master_link_status:up
master_last_io_seconds_ago:9
master_sync_in_progress:0
slave_repl_offset:281
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```
```
docker exec -it redis redis-cli -h 192.168.1.14 -a ZDU0NTlkNDY5NWZi info replication 
# Replication
role:master
connected_slaves:1
slave0:ip=192.168.1.15,port=6379,state=online,offset=295,lag=1
master_repl_offset:295
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:294
```
```
docker exec -it redis redis-cli -h 192.168.1.14 -a ZDU0NTlkNDY5NWZi set Test_Write_key www.shangtv.cn 		#创建数据
OK
```
```
docker exec -it redis redis-cli -h 192.168.1.14 -a ZDU0NTlkNDY5NWZi get Test_Write_key 
www.shangtv.cn
```