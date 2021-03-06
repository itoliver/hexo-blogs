---
title: registry v2搭建
date: 2018-04-08 11:16:20
tags: [docker, docker-registry, linux自动化运维]
---

前言
新版 registry v2对镜像存储格式进行了重新设计，并且和旧版还不兼容。registry v2是由go语言开发，docker从1.6版本开始支持registry v2，之前python开发的老版registry在网上已被标为废弃了（没有维护更新，但也可以用）。

之前在测试环境搭建了一个老版的registry，用了也比较久了。为了跟上技术的脚步，也准备今后使用新版registry v2。由于对旧版是不兼容的，所以之前仓库的数据目录还不能直接拿来挂载，只好重新做个新的，镜像只好等以后慢慢再放上去了。下面对我这次配置的步骤简单的介绍一下。
<!--more-->
# 服务器环境
本次使用centos7.3的操作系统，服务器IP假设为：192.168.0.100
预先装好docker服务，操作如下：

# 添加docker.repo安装源，写入文件
```
tee /etc/yum.repos.d/docker.repo<<EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```
# 安装docker
```
yum install docker-engine -y
systemctl enable docker
systemctl start docker
```
# 1. 获取最新的registry的容器,了解到目前最新版为2.4.1，于是直接使用docker pull命令从公用仓库去拉即可
```
docker pull registry:2.4.1
```
# 2. 运行registry:2.4.1容器
这里需要注意的是新registry仓库数据目录的位置。之前老版的位置是/tmp/registry，hub.docker.com上的演示命令里写的是/tmp/registry-dev，其实这个不对。试验证明，新registry的仓库目录是在/var/lib/registry，所以运行时挂载目录需要注意。
```
docker run -d -p 5000:5000 --restart=always \
-v /opt/registry-var/:/var/lib/registry/ \
registry:2.4.1
```
-v选项指定将/opt/registry-var/目录挂载给/var/lib/registry/
当使用curl http://192.168.0.100:5000/v2/_catalog能看到json格式的返回值时，说明registry已经运行起来了。

# 3. 修改配置文件以指定registry地址
上面registry虽然已经运行起来了，但是如果想用push命令上传镜像是会报错的，需要在配置文件中指定registry的地址。在/lib/systemd/system/docker.service文件中添加一下配置：
--insecure-registry 192.168.0.100:5000'

为了配置简单，省去安全相关的配置，这里使用--insecure-registry选项修改配置文件后，一定要重启docker服务才能生效，
```
systemctl restart docker
```
这时再push就可以上传镜像到所搭建的registry仓库了。需要注意的是，上传前要先给镜像tag一个192.168.0.100:5000/为前缀的名字，这样才能在push的时候存到私库。
```
docker tag docker.io/registry:2.4.1 192.168.0.100:5000/registry:2.4.1
docker push 192.168.0.100:5000/registry:2.4.1
```
# 4. 配置带用户权限的registry
到上面为止，registry已经可以使用了。如果想要控制registry的使用权限，使其只有在登录用户名和密码之后才能使用的话，还需要做额外的设置。

registry的用户名密码文件可以通过htpasswd来生成：
```
mkdir /opt/registry-var/auth/
docker run --entrypoint htpasswd registry:2.4.1 -Bbn felix felix  >> /opt/registry-var/auth/htpasswd
```
上面这条命令是为felix用户名生成密码为felix的一条用户信息，存在/opt/registry-var/auth/htpasswd文件里面，文件中存的密码是被加密过的。
使用带用户权限的registry时候，容器的启动命令就跟上面不一样了，将之前的容器停掉并删除，然后执行下面的命令：
```
docker run -d -p 5000:5000 --restart=always \
-v /opt/registry-var/auth/:/auth/ \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-v /opt/registry-var/:/var/lib/registry/ \
registry:2.4.1
```
这时，如果直接想查看仓库信息、pull或push都会出现权限报错。必须先使用docker login 命令来登录私有仓库：
```
docker login 192.168.0.100:5000
```
根据提示，输入用户名和密码即可。如果登录成功，会在/root/.docker/config.json文件中保存账户信息，这样就可以继续使用了。
