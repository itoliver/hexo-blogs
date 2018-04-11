---
title: centos7部署saltstack自动化平台
date: 2018-04-11 16:14:54
description: SaltStack是(C/S)架构的集中化管理平台，SaltStack基于Python语言，采用zeromq消息队列进行通信(tcp,ipc)。
categories: Linux自动化建设
tags: [linxu, shell, linux自动化运维, saltstack]
copyright: true
password:
comments:
---
## 部署环境
系统：centos7.3
centos7默认防火墙是firewall，修改为iptables(方法自行百度)
salt-master:192.168.1.100
salt-minion-1:192.168.1.200
salt-minion-2:192.168.1.300

# master部署
# 1.1 查看centos的版本和内核版本以及安装配置阿里云yum源
```
cat /etc/redhat-release
CentOS Linux release 7.3.1611 (Core)
#wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
# 1.2 安装epel-release和salt-master工具包
```
yum install epel-release -y
yum install salt-master -y
```
# 1.3 配置saltstatck开机自启动服务
```
systemctl enable salt-master.service
```
# 1.4 启动saltstack master服务
```
systemctl start salt-master
```
# 1.5 检查saltstack端口及进程的运行状态
* 4505是saltstack管理服务器发送命令消息的端口，4506是消息返回时所用的端口，saltstack一般是启动多个进程并发工作的
```
netstat -ntlp|grep python
tcp 0 0 120.76.40.16:4505 0.0.0.0:* LISTEN 4916/python 
tcp 0 0 120.76.40.16:4506 0.0.0.0:* LISTEN 4936/python
```
```
ps -aux |grep salt-master|grep -v grep
root 4906 0.0 0.0 314468 27816 ? Ss 10:47 0:00 /usr/bin/python /usr/bin/salt-master
root 4915 0.3 0.1 414628 37948 ? Sl 10:47 0:36 /usr/bin/python /usr/bin/salt-master
root 4916 0.0 0.0 396528 23580 ? Sl 10:47 0:00 /usr/bin/python /usr/bin/salt-master
root 4917 0.0 0.0 396528 25920 ? Sl 10:47 0:00 /usr/bin/python /usr/bin/salt-master
root 4920 0.0 0.0 314468 22936 ? S 10:47 0:00 /usr/bin/python /usr/bin/salt-master
root 4923 0.0 0.0 1057776 32016 ? Sl 10:47 0:01 /usr/bin/python /usr/bin/salt-master
root 4924 0.0 0.1 1205240 34072 ? Sl 10:47 0:01 /usr/bin/python /usr/bin/salt-master
root 4928 0.0 0.1 1205976 34252 ? Sl 10:47 0:01 /usr/bin/python /usr/bin/salt-master
root 4931 0.0 0.1 1206252 34200 ? Sl 10:47 0:01 /usr/bin/python /usr/bin/salt-master
root 4933 0.0 0.0 1057964 32280 ? Sl 10:47 0:01 /usr/bin/python /usr/bin/salt-master
root 4936 0.0 0.0 691476 23472 ? Sl 10:47 0:00 /usr/bin/python /usr/bin/salt-master
```
# 1.6 配置iptables防火墙)（ps：注意selinux状态，阿里云服务器默认是disabled）
```
vim /etc/systconfig/iptables加入两行
-A INPUT -p tcp -m state --state NEW -m tcp --dport 4505 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 4506 -j ACCEPT
```
```
systemctl restart iptables
```
# 配置salt-minion
# 2.1 查看centos的版本和内核版本以及安装配置阿里云yum源
```cat /etc/redhat-release
CentOS Linux release 7.3.1611 (Core)
```
安装centos7 yum源
```
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```
# 2.2 安装epel-release和salt-minion工具包
```
yum install epel-release -y
yum install salt-minion -y
```
# 2.3 配置minion配置
master参数指定master 的ip (或者主机名)，必配参数，如果minion启动时不能解析到master 主机，启动会失败；
```
sed -i 's/#master: salt/master: 192.168.1.100/g' /etc/salt/minion
```
id参数设置salt-minion名，默认未设置，minio名取主机hostname中设定的主机名
```
sed -i 's/#id:/id: 192.168.200/g' /etc/salt/minion
```
# 2.4 配置saltstatck开机自启动服务
```
systemctl enable salt-minion
```
# 2.5 启动saltstack minion服务
```
systemctl start salt-minion
```
# 其他minion同样配置 
# saltstack具体操作
```
salt-key -L                                #查看salt-key
Accepted Keys:
salt-minion-01
salt-minion-02
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```
```
salt-key -A -y                      #添加salt-key
The following keys are going to be accepted:
Unaccepted Keys:
salt-minion-01
salt-minion-02
Key for minion salt-minion-01 accepted.
Key for minion salt-minion-02 accepted.
```
```
salt-key -L                                #查看salt-key
Accepted Keys:
salt-minion-01
salt-minion-02
Denied Keys:
Unaccepted Keys:
Rejected Keys:
```
```
salt salt-minion* test.ping        #简单测试
salt-minion-01:
    True
salt-minion-02:
    True
```
```
salt salt-minion* cmd.run 'uname -r'        #运行linux命令
salt-minion-01:
    3.10.0-327.el7.x86_64
salt-minion-02:
    3.10.0-327.el7.x86_64
```