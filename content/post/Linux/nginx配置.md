---
# 常用定义
title: "Nginx配置" # 标题
date: 2021-10-02T00:34:44+08:00    # 创建时间
lastmod: 2021-10-02T00:34:44+08:00 # 最后修改时间
tags: ['Linux']
categories: ['Linux']  # 分类
---



## 安装

`centOs: yum install nginx`

目录： 

~~~linux
常用指令
目录： 
/etc/nginx
配置文件
/etc/nginx/nginx.conf
配置目录
/etc/nginx/conf.d/xxx.conf
指令
systemctl  restart nginx.service
nginx -t
nginx -s relad
~~~



## 配置文件

~~~nginx
## 代理/etc/nginx/conf.d/xxx.conf
server {
    listen 80;
    server_name jtbit.cn;
    rewrite ^(.*)$  https://$host$1 permanent;
}
## 
server {
## 不打日志的时候，linux提供了 无限0区域和null区域 待以后出文章
access_log  /dev/null;
#SSL 访问端口号为 443
    listen 443 ssl;
 #填写绑定证书的域名
    server_name jtbit.cn;
 #证书文件名称
    ssl_certificate /etc/nginx/ssl/1_jtbit.cn_bundle.crt;
 #私钥文件名称
    ssl_certificate_key /etc/nginx/ssl/2_jtbit.cn.key;
    ssl_session_timeout 5m;
 #请按照以下协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
 #请按照以下套件配置，配置加密套件，写法遵循 openssl 标准。
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
    ssl_prefer_server_ciphers on;
    location / {
        # 代理普通路径
        #root /docker/laravel7/www/blog/public;
        #try_files $uri $uri/ /index.php?$query_string;
        # 对方必须有监控  一般是 nginx php tomcat 阿帕奇等
        proxy_pass http://172.18.0.21:80/;
        include proxy.conf;
        ## 注意配置日志
        access_log  /var/logs/project/boot_access_log.log;
        error_log  /var/logs/project/boot_error.log;
        ## body大小限制  默认是？
		client_max_body_size 20M;

     }
    ## 配置404
    error_page 404 /404.html;
        location = /404.html {
        }
	##50x
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }


}
~~~



## 定义负载均衡服务器

~~~nginx
## 并且设置变量  可以百分比模式  也可以均匀模式待补充
upstream demo{  
    server localhost:1111;
    server localhost:2222;
}
server{
    listen 8080;
    location / {
           # root   html;
            #index  http://localhost:8080/;
        	## 使用变量进行反向代理，变量有负载均衡设定
			proxy_pass http://demo; 
    }
}
~~~





## 代理配置

~~~nginx
# proxy.conf文件
proxy_connect_timeout 300s;
proxy_send_timeout   300s;
proxy_read_timeout   300s;
proxy_buffer_size    32k;
proxy_buffers     4 32k;
proxy_busy_buffers_size 64k;
proxy_redirect     off;
proxy_hide_header  Vary;
proxy_set_header   Accept-Encoding '';
proxy_set_header   Host   $http_host;
proxy_set_header   Referer $http_referer;
proxy_set_header   Cookie $http_cookie;
proxy_set_header   X-Real-IP  $remote_addr;
proxy_set_header   X-Forwarded-For $http_x_forwarded_for;
proxy_set_header   X-Forwarded-Proto $scheme;
~~~

## 代理匹配

~~~nginx
## 注意 /xxx/ 和 /xxx  是有区别的
location /graduate/ {
        #root /docker/laravel7/www/blog/public;
        #try_files $uri $uri/ /index.php?$query_string;
        #root /graduation;
        client_max_body_size 20M;
        proxy_pass http://172.18.0.29:80/graduate/;
        include proxy.conf;
}

~~~

## 代理PHP

准备代理进容器的时候，php可以采用静动分离的模式进行代理，将web目录挂载进入容器，phpfpm进行处理

>  （注意**阿帕奇其实可以更方便**，这只是方案的一种，fpm注意开启0.0.0.0监听）

- 静态文件直接   ` location /` 代理进web目录
- 动态文件（php）代理进挂载在容器的web目录，交给php-fpm 处理

~~~nginx
server {
    listen 81;
    server_name levani.cn;
	
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";
	## 设置主页
    index index.html index.htm index.php;
        access_log  /dev/null;
        error_log  /var/logs/project/error.log;
    charset utf-8;
	## 静态文件
    location / {
        # 挂载的外部目录
        root /docker/laravel7/www/blog/public;
        # 转到php匹配
        try_files $uri $uri/ /index.php?$query_string;
    }
	
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    #error_page 404 /index.php;
	## 核心  匹配的PHP文件， 注意
    location ~ \.php$ {
        ## 代理进docker内的路径 文件是在容器内找的
        # 因为php-fpm是由socket通讯，容器内部没有指向的目录，是由通讯方指定的
        root /var/www/blog/public; #对应容器内的  /var/www/blog/public;
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        # 核心路径 $document_root/$fastcgi_script_name 对应就是  /var/www/blog/public/匹配路径（记得）
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; #$realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

~~~





[负载均衡]: https://blog.csdn.net/qq_36478297/article/details/112138716