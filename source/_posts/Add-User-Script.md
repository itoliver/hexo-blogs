---
title: Add User Script
date: 2018-04-04 18:39:32
tags: [linux, shell, linux自动化运维]
---
前言
可用于服务器自动添加用户，并发送邮件
实现shell自动化建设
<!--more-->
```+shell
#!/bin/bash
password="password"
ip=`ifconfig eth1 |grep inet |awk '{printf "IP:"}''{print $2}'`
echo "请输入要创建的用户名："
read username
echo "您输入的用户名为: $username"
egrep "^servergroups" /etc/group >& /dev/null
if [ $? -ne 0 ]
then
    groupadd servergroups
else
    echo "已存在servergroups组"
fi

egrep "^$username" /etc/passwd >& /dev/null
if [ $? -ne 0 ]
then
    useradd $username && \
    usermod -aG servergroups $username && \
    echo $username | passwd --stdin $username
    chage -d 0 $username
    echo "创建 $username 用户成功"
    #echo "密码为:$password"
    id $username
    text="用户名：$username\n密码为：$username\n$ip\n注意：第一次登陆必须要修改密码!!!!"
    echo -e "$text" | mail -s "$username,服务器用户创建成功，请及时修改密码!!" $username@qq.com
else
    echo "$username 已存在"
    id $username
fi
```