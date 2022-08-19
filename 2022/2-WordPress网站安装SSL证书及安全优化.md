最近我的博客上线了，接下里我准备给它安装ssl证书。此外，我在网站的访问日志中，发现了大量恶意的请求，主要是尝试登录、尝试执行恶意脚本、获取敏感信息等。虽然WordPress核心总体来说已经足够安全，但是我还是决定禁用xmlrpc，隐藏后台登录入口，以提高网站的安全性。

## 安装免费的ssl证书-Let's Encrypt

Certbot是一个开源免费的工具，主要功能是为网站自动安装基于Let’s Encrypt服务的SSL证书。

- 安装Certbot

```bash
apt install certbot
apt install python3-certbot-nginx
```

- 验证网站所有权，生成证书文件

```bash
certbot certonly --nginx
# 或者让Certbot自动编辑您的nginx配置
# certbot --nginx 
```
*默认生成证书的目录 `/etc/letsencrypt/live/$domain/`*

- 编辑nginx配置，手动配置证书

```
server {
    listen 80;
    server_name  kxler.com www.kxler.com;
    # http强制到https
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;

    server_name  kxler.com www.xkler.com;
    root /workspace/wordpress;

    # ssl证书
    ssl_certificate           /etc/letsencrypt/live/kxler.com/fullchain.pem;
    ssl_certificate_key       /etc/letsencrypt/live/kxler.com/privkey.pem;
    ssl_session_timeout  30m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';

    #以下省略...
}
```

重启nginx
```bash
# 测试配置是否正确
nginx -t
# 重启nginx
nginx -s reload
```

此时网站后台无法使用https域名登录，需要使用IP或http登录到wordpress后台，在`设置>常规`修改站点URL配置为https地址`https://www.kxler.com`。

- 关于自动续订

Let's Encrypt颁发的是短期证书（90 天），需要3个月内至少续订一次证书。

大多数Certbot安装都带有开箱即用的自动续订功能，Cerbot使用 `certbot renew` 完成自动续订，生成的续订定时任务在 `/etc/cron.d/certbot`。

如果您仍不确定，可以按照官网手动配置自动续订。
[Certbot自动续订](https://eff-certbot.readthedocs.io/en/stable/using.html#automated-renewals)

## 禁用xmlrpc.php

最近查看nginx访问日志，发现了大量关于xmlrpc.php的恶意请求。经查资料得知，XML-RPC是支持WordPress与其他系统之间通信的规范，如今已被REST API取代。所以决定禁用它。

禁用方式有很多种，这里选择修改nginx配置文件，在wordpress.conf配置中添加
```
# 务必加在 location ~ \.php${...} 前面的位置
location ~* ^/xmlrpc.php$ {
    return 403;
}
```

重启nginx
```bash
# 测试配置是否正确
nginx -t
# 重启nginx
nginx -s reload
```

## 隐藏登录入口

此外，在nginx访问日志，也发现了大量尝试登录的请求，这里直接修改登录入口，以提高安全性。

首先安装插件`WPS Hide Login`，然后在后台配置登录入口。

另外，如果觉得有必要还可安装限制登录频率的插件。
