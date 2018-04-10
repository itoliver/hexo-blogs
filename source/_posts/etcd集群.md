---
title: etcd集群
date: 2018-04-08 10:47:45
tags: [docker, linux自动化运维, etcd]
copyright: true
password:
---
前言
Etcd:一个键值存储仓库，主要用于配置共享和服务发现。
优点：
	简单：支持 curl 方式的用户 API (HTTP+JSON)
	安全：可选 SSL 客户端证书认证
	快速：单实例可达每秒 1000 次写操作
	可靠：使用 Raft 实现分布式

本文主要阐述集群etcd的部署
<!--more-->
安装etcd可以通过源码编译安装，也可以用yum安装，这里实验用yum安装
Configure epel yum
```
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && rpm -ivh epel-release-latest-7.noarch.rpm
```
install etcd
```
yum install -y etcd
```
configure host
```
echo "etcd1 192.168.1.100" >> /etc/hosts
echo "etcd2 192.168.1.200" >> /etc/hosts
```
```
IP=$(ifconfig  eth1 |awk  -F '[: ]+' 'NR==2{print $3}')
IP1=192.168.1.100
IP2=192.168.1.200
HOST="etcd1=http://192.168.1.100:2380,etcd2=http://192.168.1.200:2380"
```
```
function node1{
#Configure the node1 etcd file
sed -i 's#\#ETCD_LISTEN_PEER_URLS="http://localhost:2380"#ETCD_LISTEN_PEER_URLS="http://${IP1}:2380"#g' /etc/etcd/etcd.conf
sed -i 's#ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"#ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"#g' /etc/etcd/etcd.conf
sed -i 's#ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"#ETCD_ADVERTISE_CLIENT_URLS="http://${IP1}:2379"#g' /etc/etcd/etc.conf
sed -i 's#\#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"#ETCD_INITIAL_CLUSTER="${HOST}"#'g

}

function node2{
#Configure the node2 etcd file
sed -i 's#\#ETCD_LISTEN_PEER_URLS="http://localhost:2380"#ETCD_LISTEN_PEER_URLS="http://${IP2}:2380"#g' /etc/etcd/etcd.conf
sed -i 's#ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"#ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"#g' /etc/etcd/etcd.conf
sed -i 's#ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"#ETCD_ADVERTISE_CLIENT_URLS="http://${IP2}:2379"#g' /etc/etcd/etc.conf
sed -i 's#\#ETCD_INITIAL_CLUSTER="default=http://localhost:2380"#ETCD_INITIAL_CLUSTER="${HOST}"#g' /etc/etcd/etc.conf
}

function start{
	systemctl enable etcd
	systemctl restart etcd
	systemctl status etcd
}

if [ "$IP"=="$IP1" ];then
	node1
	start
else
	node2
	start
fi
```

look etcd node list
```
etcdctl member list
```
look etcd node status
```
etcdctl  cluster-health
```