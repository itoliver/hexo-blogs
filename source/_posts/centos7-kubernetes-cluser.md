---
title: centos7部署kubernetes-cluser
date: 2018-04-11 16:29:18
description:
categories: [Docker, Linux自动化建设]
tags: [docker, kubernetes, linux自动化运维]
copyright: true
password:
comments:
---
前言
Kubernetes是Google开源的容器集群管理系统，其提供应用部署、维护、 扩展机制等功能，利用Kubernetes能方便地管理跨机器运行容器化的应用，是Docker分布式系统的解决方案。k8s里所有的资源都可以用yaml或Json定义。
K8s基本概念

Master
Master节点负责整个集群的控制和管理，所有的控制命令都是发给它，上面运行着一组关键进程：
kube-apiserver：提供了HTTP REST接口，是k8s所有资源增删改查等操作的唯一入口，也是集群控制的入口。
kube-controller-manager：所有资源的自动化控制中心。当集群状态与期望不同时，kcm会努力让集群恢复期望状态，比如：当一个pod死掉，kcm会努力新建一个pod来恢复对应replicas set期望的状态。
kube-scheduler：负责Pod的调度。
实际上，Master只是一个名义上的概念，三个关键的服务不一定需要运行在一个节点上。

Node
Node是工作负载节点，运行着Master分配的负载（Pod），但一个Node宕机时，其上的负载会被自动转移到其他Node上。其上运行的关键组件是：
kubelet：负责Pod的生命周期管理，同时与Master密切协作，实现集群管理的基本功能。
kube-proxy：实现Service的通信与负载均衡机制的重要组件，老版本主要通过设置iptables规则实现，新版1.9基于kube-proxy-lvs 实现。
Docker Engine：Docker引擎，负责Docker的生命周期管理。
<!--more-->
# 安装前准备
# 1.操作系统详情
需要三台主机，都最小化安装 centos7.3,并update到最新
```
[root@master ~]# cat /etc/redhat-release 
CentOS Linux release 7.3.1611 (Core)
角色       主机名         IP
Master  master  192.168.1.14
node1  slave-1  192.168.1.15
node2  slave-2  192.168.1.16
```
# 2.在每台主机上关闭firewalld改用iptables
输入以下命令，关闭firewalld
```
systemctl stop firewalld.service #停止firewall 
systemctl disable firewalld.service #禁止firewall开机启动
yum install iptables-* -y
systemctl start iptables.service
systemctl enable iptables.service
```
# 3.安装ntp服务
```
yuminstall -y ntp wget net-tools 
systemctl start ntpd systemctl enable ntpd
```
# 安装配置
注：kubernetes，etcd等已经进去centos epel源，可以直接yum安装（需要安装epel-release）

# 1.安装Kubernetes Master
使用以下命令安装kubernetes 和 etcd
```
yum install -y kubernetes etcd
```
编辑/etc/etcd/etcd.conf 使etcd监听所有的ip地址，确保下列行没有注释，并修改为下面的值
```
cat /etc/etcd/etcd.conf 
# [member] ETCD_NAME=default ETCD_DATA_DIR="/var/lib/etcd/default.etcd" 
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379" 
ETCD_INITIAL_CLUSTER="default=http://192.168.1.14:2380" 
#[cluster] 
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.14:2379"
```

编辑Kubernetes API server的配置文件 /etc/kubernetes/apiserver，确保下列行没有被注释，并为下列的值
```
cat /etc/kubernetes/apiserver
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet_port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://192.168.1.14:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS=""
```
启动etcd, kube-apiserver, kube-controller-manager and kube-scheduler服务，并设置开机自启

```
cat /script/kubenetes_service.sh
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
```
```
sh /script/kubenetes_service.sh
```
在etcd中定义flannel network的配置，这些配置会被flannel service下发到nodes:
```
etcdctl mk /centos.com/network/config '{"Network":"172.17.0.0/16"}'
```
添加iptables规则，允许相应的端口
```
iptables -I INPUT -p tcp --dport 2379 -j ACCEPT
iptables -I INPUT -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT 
```
```
iptables-save
``` 
或者写入iptables配置文件 /etc/sysconfig/iptables

查看节点信息（我们还没有配置节点信息，所以这里应该为空）

```
kubectl get nodes
NAME LABELS STATUS
```
# 2.安装Kubernetes Nodes
注：下面这些步骤应该在node1和node2上执行（也可以添加更多的node）

使用yum安装kubernetes 和 flannel
```
yum install -y flannel kubernetes
```
为flannel service配置etcd服务器，编辑/etc/sysconfig/flanneld文件中的下列行以连接到master
```
cat /etc/sysconfig/flanneld

FLANNEL_ETCD="http://192.168.1.14:2379" #改为etcd服务器的ip
FLANNEL_ETCD_PREFIX="/centos.com/network"
```
编辑/etc/kubernetes/config 中kubernetes的默认配置，确保KUBE_MASTER的值是连接到Kubernetes master API server：
```
cat /etc/kubernetes/config

KUBE_MASTER="--master=http://192.168.1.14:8080"
```
编辑/etc/kubernetes/kubelet 如下行：

node1:
```
cat /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname_override=192.168.1.15"
KUBELET_API_SERVER="--api_servers=http://192.168.1.14:8080"
KUBELET_ARGS=""
```
node2:
```
cat /etc/kubernetes/kubelet

KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname_override=192.168.1.16"
KUBELET_API_SERVER="--api_servers=http://192.168.1.14:8080"
KUBELET_ARGS=""
```
启动kube-proxy, kubelet, docker 和 flanneld services服务，并设置开机自启
```
cat /script/kubernetes_node_service.sh

for SERVICES in kube-proxy kubelet docker flanneld; do 
systemctl restart $SERVICES
systemctl enable $SERVICES
systemctl status $SERVICES 
done
```
在每个node节点，你应当注意到你有两块新的网卡docker0 和 flannel0。你应该得到不同的ip地址范围在flannel0上，就像下面这样：

node1:
```
ip a | grep flannel | grep inet
inet 172.17.11.0/16 scope global flannel0
```
node2:
```
ip a | grep flannel | grep inet
inet 172.17.60.0/16 scope global flannel0
```
添加iptables规则：

```
iptables -I INPUT -p tcp --dport 2379 -j ACCEPT
iptables -I INPUT -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
```
现在登陆kubernetes master节点验证minions的节点状态：
```
kubectl get nodes
NAME           STATUS    AGE
192.168.1.15   Ready     2h
192.168.1.16   Ready     2h
```
至此，kubernetes集群已经配置并运行了，我们可以继续下面的步骤。

# 创建 Pods (Containers)
为了创建一个pod，我们需要在kubernetes master上面定义一个yaml 或者 json配置文件。然后使用kubectl命令创建pod

```
mkdir -p /k8s/pods
cd /k8s/pods/
```
查看nginx.yaml内容如下：
```
apiVersion: v1
kind: Pod
metadata:
name: nginx
labels:
app: nginx
spec:
containers:
- name: nginx
image: nginx
ports:
- containerPort: 80
```
创建pod:
```
kubectl create -f nginx.yaml
```
此时有如下报错：
`Error from server: error when creating "nginx.yaml": Pod "nginx" is forbidden: no API token found for service account default/default, retry after the token is automatically created and added to the service account`
解决办法是编辑/etc/kubernetes/apiserver 去除 KUBE_ADMISSION_CONTROL中的SecurityContextDeny,ServiceAccount，并重启kube-apiserver.service服务：

```
cat /etc/kubernetes/apiserver
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"
```
```
systemctl restart kube-apiserver.service
```
之后重新创建pod:

```
kubectl create -f nginx.yaml
pods/nginx
```
查看pod:
```
kubectl get pod nginx
NAME READY STATUS RESTARTS AGE
nginx 0/1 Image: nginx is not ready on the node 0 34s
```
这里STATUS一直是这个，创建不成功，下面排错。通过查看pod的描述发现如下错误：
```
kubectl describe pod nginx 
Wed, 28 Oct 2015 10:25:30 +0800 Wed, 28 Oct 2015 10:25:30 +0800 1 {kubelet 192.168.1.16} implicitly required container POD pulled Successfully pulled Pod container image "gcr.io/google_containers/pause:0.8.0"
Wed, 28 Oct 2015 10:25:30 +0800 Wed, 28 Oct 2015 10:25:30 +0800 1 {kubelet 192.168.1.16} implicitly required container POD failed Failed to create docker container with error: no such image
Wed, 28 Oct 2015 10:25:30 +0800 Wed, 28 Oct 2015 10:25:30 +0800 1 {kubelet 192.168.1.16} failedSync Error syncing pod, skipping: no such image
Wed, 28 Oct 2015 10:27:30 +0800 Wed, 28 Oct 2015 10:29:30 +0800 2 {kubelet 192.168.1.16} implicitly required container POD failed Failed to pull image "gcr.io/google_containers/pause:0.8.0": image pull failed for gcr.io/google_containers/pause:0.8.0, this may be because there are no credentials on this request. details: (API error (500): invalid registry endpoint "http://gcr.io/v0/". HTTPS attempt: unable to ping registry endpoint https://gcr.io/v0/
v2 ping attempt failed with error: Get https://gcr.io/v2/: dial tcp 173.194.72.82:443: i/o timeout
```
这里可能会遇到pod状态一直处于Penning的问题，此时可以通过kubectl describe pods/pod-name来查看pod信息，如果没有出错信息，那么Minion一直处于下载镜像中，下载好之后pod即会成功启动。

从网上找到 pause:0.8.0 的镜像，然后再每个node上导入镜像：

请在境外docker服务器执行 docker pull 命令下载镜像
```
gcr.io/google_containers/pause:latest
gcr.io/google_containers/pause:1.0
gcr.io/google_containers/pause:0.8.0
```
再用导出镜像
```
docker save -o pause.tar gcr.io/google_containers/pause
gzip pause.tar
```
最后把这个包放到 kubernetes 环境所有的 docker 服务器上
```
docker load -i pause.tar.gz
```
在执行以下命令即可成功创建pod
```
kubectl create -f nginx.yaml
pods/nginx
```
查看pod
```
kubectl get pod nginx
NAME READY STATUS RESTARTS AGE
nginx 1/1 Running 0 2min
```
前往nodes节点上查看docker images
```
docker images
REPOSITORY                                            TAG                 IMAGE ID            CREATED             SIZE
registry.access.redhat.com/rhel7/pod-infrastructure   latest              34d3450d733b        10 weeks ago        205 MB
gcr.io/google_containers/pause                        0.8.0               bf595365a558        2 years ago         241.7 kB
``` 
