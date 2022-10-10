WordPress作为全球最流行的博客系统，拥有丰富的主题和插件，是搭建博客的不二选择。下面将开始介绍从零搭建个人博客的步骤。

## 技术选型
- debian：版本号buster，由于华为云的公共镜还没出debian11，目前只有采用debian10
- nginx：当前最新稳定版1.22.0
- php：版本7.4，考虑到插件可能有php8不兼容，最好还是php7.4
- mysql：版本5.7，mysql8采用了新的认证插件，可能旧的客户端驱动不兼容，还是选择了5.7

## 前置操作
- 购买云服务器，这里我选择了华为云

- 购买一个域名，我的域名是 www.kxler.com

- 打开terminal或powershell或其他客户端工具登录远程服务器
`ssh root@your ip`

## 安装基础软件

如果采用的是华为云服务器，替换源镜像地址和设置时区可以忽略。因为华为云服务器默认使用了华为云debian镜像，并且设置了上海时区。
```bash
# 使用阿里云debian镜像
cp /etc/apt/sources.list /etc/apt/sources.list.bak \
    && sed -i -E 's/(deb|security).debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list

# 设置时区
ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' > /etc/timezone
```

```bash
# 常规操作，更新和升级软件库
apt update -y && apt upgrade -y
```

```bash
# 安装常用工具
apt install -y vim git wget curl \
    && apt install -y apt-utils telnet net-tools procps iputils-ping lsb-release
```

## 安装Nginx

```bash
# 添加nginx apt存储库
# 也可以使用apt-key add添加源，不过debian11将要废弃
echo "deb http://nginx.org/packages/debian/ $(lsb_release -sc) nginx" >> /etc/apt/sources.list \
    && echo "deb-src http://nginx.org/packages/debian/ $(lsb_release -sc) nginx" >> /etc/apt/sources.list

apt install -y gnupg2 \
    && curl -s https://nginx.org/keys/nginx_signing.key | gpg --no-default-keyring --keyring gnupg-ring:/etc/apt/trusted.gpg.d/nginx.gpg --import \
    && chown _apt /etc/apt/trusted.gpg.d/nginx.gpg
```

```bash
# 安装
apt update -y
# 如果此时使用 apt search ^nginx$ 可以看到nginx已经是最新的稳定版本了
apt install nginx
```

```bash
# 确认安装成功
nginx -v
```

## 安装PHP

目前debian11的默认php版本7.4，而debian10默认php版本7.3。我选择的debian10，这里需要添加php apt储存库。
```bash
# 添加php apt存储库
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/php.list

wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
```

```bash
apt update -y
# 此时使用 apt search php7.4 确认已有php7.4版本
apt install -y php7.4 php7.4-fpm
```

```bash
# 确认安装成功
php -v
php-fpm7.4 -v
```

## 安装MySQL

```bash
# 下载包
cd /tmp && wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-server_5.7.39-1debian10_amd64.deb-bundle.tar
# 解压
tar -xvf mysql-server_5.7.39-1debian10_amd64.deb-bundle.tar
# 安装依赖
apt install -y libaio1 libatomic1 libmecab2 libnuma1 psmisc libncurses6
# 依次安装，期间有提示输入和确认密码
dpkg -i mysql-common_5.7.39-1debian10_amd64.deb \
    && dpkg -i mysql-community-client_5.7.39-1debian10_amd64.deb \
    && dpkg -i mysql-client_5.7.39-1debian10_amd64.deb \
    && dpkg -i mysql-community-server_5.7.39-1debian10_amd64.deb \
    && dpkg -i mysql-server_5.7.39-1debian10_amd64.deb
# 配置文件 /etc/mysql/mysql.conf.d/mysqld.cnf
```

## 启动服务
```bash
# 启动 nginx php-fpm mysql
service nginx start
service php7.4-fpm start
service mysql start
# 使用 ps aux 查看进程是都启动
```

## 安装wordpress源码
这里将wordpress源码安装到`/workspace/wordpress`
```bash
cd /tmp && wget https://cn.wordpress.org/latest-zh_CN.tar.gz \
    && mkdir /workspace \
    && tar -xvf latest-zh_CN.tar.gz -C /workspace
```

## 配置
#### 配置nginx server

**调整nginx.conf如下：**

<details>
<summary><u>/etc/nginx/nginx.conf</u></summary>
<pre>
<blockcode>
#user  nginx;
user  www-data;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #long time
    #check_shm_size 5M;
    # Allow the server to close the connection after a client stops responding.
    reset_timedout_connection on;
    client_header_timeout 15;
    # Send the client a "request timed out" if the body is not loaded by this time.
    client_body_timeout 10;
    # If the client stops reading data, free up the stale client connection after this much time.
    send_timeout 15;
    # Timeout for keep-alive connections. Server will close connections after this time.
    keepalive_timeout 30;
    # Number of requests a client can make over the keep-alive connection.
    keepalive_requests 30;

    client_body_buffer_size 128k;
    client_max_body_size 10m;
    proxy_read_timeout 180s;

    # Compression.
    gzip on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml;
    gzip_disable "msie6";

    # Sendfile copies data between one FD and other from within the kernel.
    sendfile on;
    # Don't buffer data-sends (disable Nagle algorithm).
    tcp_nodelay on;
    # Causes nginx to attempt to send its HTTP response head in one packet,  instead of using partial frames.
    tcp_nopush on;

    # Hide web server information
    # server_tokens off;
    # server_info off;
    # server_tag off;

    # redirect server error pages to the static page
    error_page 404             /404.html;
    error_page 500 502 503 504 /50x.html;

    include /etc/nginx/conf.d/*.conf;
}
</blockcode>
</pre>
</details>


**新增server配置如下：**

<details>
<summary><u>/etc/nginx/conf.d/wordpress.conf</u></summary>
<pre>
<blockcode>
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;

    server_name  kxler.com www.kxler.com;
    root /workspace/wordpress;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    fastcgi_connect_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_send_timeout 300;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location / {
        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;

        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        # 取决于配置文件/etc/php/7.4/fpm/pool.d/www.conf中listen是unix socket还是tcp
        # fastcgi_pass   127.0.0.1:9000;
        fastcgi_pass   unix:/run/php/php7.4-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|mp3|mp4|ico|woff|woff2|ttf)$ {
        expires      30d;
        log_not_found off;
        access_log off;
        }

    location ~ .*\.(js|css)?$ {
        expires      12h;
        access_log off;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    access_log  /var/log/nginx/access_wordpress-blog.log;
    error_log  /var/log/nginx/error_wordpress-blog.log error;
}
</blockcode>
</pre>
</details>

**检查&重启**
```bash
# 检查配置是否正确
nginx -t
# 重启nginx
nginx -s reload
```

#### MySQL建库、新增用户并授权

*使用之前的输入的密码登录mysql root账户*
`mysql -uroot -p`

执行以下sql创建数据库和用户
```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_general_ci;
# 新增用户，用户名wordpress 密码123456。请自行替换成更加复杂的密码
CREATE USER 'wordpress'@'%' IDENTIFIED BY "123456";
GRANT ALL ON wordpress.* TO 'wordpress'@'%';
```

**准备就绪，开始访问网站，下面记录首次访问网站报的错与解决方法**

## 踩坑和埋坑

我申请的域名备案此时还没通过审核，无法用域名访问我的网站，这里直接访问云服务器的公网IP（由于前面nginx配置中已经将wordpress设置为了默认server）

#### 网站报错502
查看nginx日志发现 `connect() to unix:/run/php/php7.4-fpm.sock failed (13: Permission denied)`。原来php-fpm默认配置listen采用unix socket通信，默认用户是www-data，创建出来的的sock文件身份是www-data，而nginx默认用户是nginx，造成权限问题。
**解决：**
将`/etc/nginx/nginx.conf`的`user`改为`www-data` (前面给出的nginx配置已改)
再重启nginx：`nginx -s reload`

#### 网站提示：您的PHP似乎没有安装运行WordPress所必需的MySQL扩展
```bash
# 安装php扩展mysqli gd，额外安装wordpress官方推荐的扩展imagick curl zip dom intl
apt install -y php7.4-mysqli php7.4-gd php7.4-mbstring php7.4-curl php7.4-imagick  php7.4-zip  php7.4-dom  php7.4-intl
# 重启php-fpm
service php7.4-fpm restart
```

#### 网站提示：无法写入wp-config.php文件
一看就是权限问题，将整个wordpress源码授予用户www-data
`chown -R www-data:www-data /workspace/wordpress`

#### 调整上传最大尺寸配置
php配置 `/etc/php/7.4/fpm/php.ini`，将默认的文件上传尺寸2M调高到8M
`upload_max_filesize = 8M`

nginx配置 `/etc/nginx/nginx.conf`(前面给出的nginx配置已改)
`client_max_body_size 8m;`

## 一切就绪
- 访问网站IP，跟着wordpress安装向导，填写数据库连接配置、网站基本配置。

- 接下来就是选择主题，安装插件。

- 大约过了10天，域名备案过审后，此时在云服务商后台将域名解析到云服务器IP，再到wordpress后台 `设置>常规` 修改站点URL配置。

- 访问网站 www.kxler.com ，享受吧！

## 额外的

**头像不显示？**
wordpress头像地址默认是Gravatar，国内被qiang了，写个插件将其替换为国内镜像地址。
参考[自定义方法插件](https://github.com/kaxiluo/wordpress-plugins "自定义方法插件")

