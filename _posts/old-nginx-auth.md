---
title: 如何在nginx里设置权限
date: 2017-12-27 18:23:34
tags: 
- 运维
- nginx
category: 运维
---

## 如何在nginx里设置权限

1. 安装nginx

略

2. 配置密码

这里拿用户名为sayuri,储存文件为/etc/nginx/.htpasswd为例子

* 使用 openssl


```bash
sudo sh -c "echo -n 'sayuri': >> /etc/nginx/.htpasswd"
sudo sh -c "openssl passwd -apr1 >> /etc/nginx/.htpasswd"
#这个命令之后会让你输入和确认密码
```

* 使用apache2-utils

首先要安装apache2-utils

之后输入
```bash
sudo htpasswd -c /etc/nginx/.htpasswd sayuri
#同样会让你输入和确认密码
```

以上两种方法可以多次输入在一个文件里添加多个用户

3. 配置文件里设置
以/etc/nginx/sites-enabled/default 为例

打开之前：
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /usr/share/nginx/html;
    index index.html index.htm;

    server_name localhost;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

在需要权限的地方添加以下设置
```
auth_basic "Restricted Content";
auth_basic_user_file /etc/nginx/.htpasswd;
```

修改后：
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    root /usr/share/nginx/html;
    index index.html index.htm;

    server_name localhost;

    location / {
        try_files $uri $uri/ =404;
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

4. 检查完整性并重启
```bash
sudo nginx -t
sudo nginx -s reload
```

### 参考资料
[How To Set Up Password Authentication with Nginx on Ubuntu 14.04 ](https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-nginx-on-ubuntu-14-04)