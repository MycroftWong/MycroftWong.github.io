---
title: nginx配置多个域名
date: 2019-08-09 14:06:50
categories: 网站搭建
tags: nginx,blog,hexo,github.gitee
---

# nginx配置多个域名

新建了这个`blog`，同时上传了`github`和`gitee`，另外想发布到个人阿里云服务器，在阿里云服务器上直接拉取`github`仓库。

## 目标
在不更改原来请求的情况下，添加二级域名`blog`，转发请求到博客文件夹。

## nginx配置解析

原本的配置如下
```txt
server {
    listen 80;
    server_name mycroft.wang www.mycroft.wang;
    
    #这是新版本的Nginx转发
    return 301 https://$server_name$request_uri;
    
    # 固定写法-------------
    tcp_nodelay     on;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

server {
        listen 443;
        server_name mycroft.wang www.mycroft.wang;
        ssl on;
        root /home/www;
        index index.html index.htm;
        ssl_certificate   ...;      // 这里省略证书地址
        ssl_certificate_key  ...;   // 这里省略证书地址
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        

    location ~ \.(html|htm|gif|jpeg|jpg|css|js|png|swf|ico)$ {
        root   /home/www;
        index  index.html index.htm;
    }

    location ~ ^/$ {
        root /home/www;
    }

    location / {
        proxy_pass http://localhost:8080;
        include /home/nginx/proxy.conf;
    }
    
    error_page 404 /404.html;
    location = /404.html {
        root /home/www/error;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /home/www/error;
    }
}
```

转发`www`二级域名到`443`端口，然后在`443`端口上转发，静态文件转发到`/home/www`文件夹，动态请求转发到`8080`端口。

### 需要的工作
1. 新建一个`server`监听`80`端口，指定监听的域名`blog.mycroft.wang`，然后转发到`443`端口
2. 新建一个`server`监听`443`端口，指定监听的域名`blog.mycroft.wang`，转发到文件夹`/home/blog`
3. 过滤掉`git`相关文件夹

下面是添加的`server`，注意没有包括之前的配置
```txt
server {
    listen 80;
    server_name blog.mycroft.wang;
    
    #这是新版本的Nginx转发
    return 301 https://$server_name$request_uri;
    
    # 固定写法-------------
    tcp_nodelay     on;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}

server {
        listen 443;
        server_name blog.mycroft.wang;
        ssl on;
        root /home/blog;
        index index.html;
        ssl_certificate   ...;      // 这里省略证书地址
        ssl_certificate_key  ...;   // 这里省略证书地址
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;

        location ~ ^(.*)/\.(svn|git)/ {
            deny all;
        }
}
```

## 说明
并没有说明`nginx`使用的原理，因为我也不懂，这是之前学了一点，依葫芦画瓢弄的，关于`nginx`的使用，后面再认真学学。


## 参考文章

[如何配置nginx 同一ip,多域名,不同端口?](https://segmentfault.com/q/1010000004915921)

[在Nginx上配置多个站点](https://www.cnblogs.com/Erick-L/p/7066564.html)

[nginx忽略.svn和.git](http://linux.it.net.cn/e/server/nginx/2016/0409/21095.html)

