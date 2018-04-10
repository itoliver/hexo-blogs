---
title: docker-nginx-php部署
date: 2018-04-09 12:38:29
description:
categories: dockers
tags: [docker, nginx, php, linux自动化运维]
copyright: true
---
前言
nginx是web服务器常用的架构,是一个高性能的HTTP和反向代理服务器,也是一个IMAP/POP3/SMTP服务器。
<!--more-->
服务器环境
nignx:基于docker最新版nginx
php:7.1.5

# 1.拉取nginx和php-fpm镜像
```
docker pull nginx
```
docker pull php:7.1.5-fpm #自定义版本
```
docker pull bitnami/php-fpm
```
# 2.创建nginx数据目录
```
mkdir -p /opt/nginx/{conf,conf.d,html,log}
mkdir -p /opt/php/conf
```
# 3.创建nginx default.conf,此配置是支持go语言的反向代理
```
cat /opt/nginx/conf.d/default.conf << EOF
map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
}
upstream gameweb {
        ip_hash;
        server 192.168.1.14:3001 weight=2;
        server 192.168.1.14:3002;
}
server
{
    listen 3003;
    server_name master.localhost.com;
    error_log /var/log/nginx/gameweb_error.log debug;
    access_log /var/log/nginx/gameweb_access.log;


location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Nginx-Proxy true;
        proxy_pass http://gameweb;

        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Credentials true;
        add_header Access-Control-Allow-Headers Content-Type,Accept;
        add_header Access-Control-Allow-Methods GET;
       
    }

}

server {
        listen       80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        location / {
            index  index.html index.htm index.php;
            autoindex  off;
        }
        location ~ \.php(.*)$ {
            root           /usr/share/nginx/html/;
            fastcgi_pass   php:9000;
            fastcgi_index  index.php;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
        }
}
EOF
```
# 4.配置php测试文件
```
tee /opt/nginx/html/index.php <<EOF
<?php
echo "TEST PAGE"
?>
EOF

# 5.启动php容器
```
docker run -tid \
-p 9000:9000 \
--name php-fpm \
--restart=always \
--privileged=true \
-v /opt/php/conf/:/bitnami/php/conf/ \
-v /opt/nginx/html:/usr/share/nginx/html \
bitnami/php-fpm
```
# 6.启动nginx容器
```
docker run -tid \
--name nginx \
--restart=always \
-p 80:80 \
--privileged=true \
-v /opt/nginx/conf.d:/etc/nginx/conf.d \
-v /opt/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /opt/nginx/html:/usr/share/nginx/html \
-v /opt/nginx/log:/var/log/nginx \
--link php-fpm:php \
nginx
```