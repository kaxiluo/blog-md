最近使用自己编写的fastapi的脚手架开发了一个项目，下面分享部署fastapi的过程。

- venv： Python创建虚拟环境的工具。

- uvicorn： Python ASGI Web服务器。轻量级，不具备进程监控。

- gunicorn： 用于UNIX的Python WSGI Web服务器。可以用来管理Uvicorn，充当进程管理器，Gunicorn的功能齐全且成熟。

本教程环境说明：操作系统 debian10； 项目 [kaxiluo/fastapi-skeleton](https://github.com/kaxiluo/fastapi-skeleton) 

## 安装python虚拟环境和项目依赖

```bash
# 创建python虚拟环境
python3 -m venv venv-fastapi-demo
# 进入虚拟环境
source ./venv-fastapi-demo/bin/activate

# 进入项目根目录
cd /path/to/fastapi-demo/

# 安装项目依赖
pip install -r requirements.txt
# 可能中途报错，提示缺少库或请升级pip
# 升级pip
python -m pip install --upgrade pip
# 安装依赖
apt install build-essential libssl-dev libffi-dev python3-dev
# 再次执行安装项目依赖命令
pip install -r requirements.txt
```

## 启动

- 方式一

使用uvicorn启动，一般用于开发环境

```bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

- 方式二

安装gunicorn，并启动

```bash
pip install gunicorn

gunicorn main:app --workers 2 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8080
```

这些启动参数也可以用一个配置文件管理，了解更多功能参数请运行 `gunicorn -h`

- 方式三

以守护进程方式运行gunicorn

```bash
nohup gunicorn main:app --workers 2 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8080 > /dev/null &

# 或者
# 其实gunicorn提供了-D参数
gunicorn main:app -D --workers 2 --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8080
```

**停止服务和重启**

首先找到gunicorn进程PID，然后通过信号量停止或重启服务

```bash
# 查看主进程ID
pstree -ap | grep gunicorn
```

停止
```bash
# 正常停止主进程及其子进程
kill -TERM pid
```

重启
```bash
kill -HUP pid
```

以上方式都不适合生产环境，下面是终极方案。

## 加入系统服务运行，配置nginx反向代理，设置开机启动

1. 新建gunicorn服务文件 `/etc/systemd/system/gunicorn.service`：

```bash
[Unit]
Description=gunicorn - python http server
After=network.target

[Service]
Type=forking
PIDFile=/var/run/gunicorn.pid
# 项目根目录
WorkingDirectory=/path/to/fastapi-demo
# gunicorn启动命令
ExecStart=/path/to/venv-fastapi-demo/bin/python3 /path/to/venv-fastapi-demo/bin/gunicorn main:app -D --pid /var/run/gunicorn.pid --workers 2 --worker-class uvicorn.workers.UvicornWorker --bind 127.0.0.1:8080
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s TERM $MAINPID

[Install]
WantedBy=multi-user.target
```

2. 加载新的服务配置文件

```bash
systemctl daemon-reload
```

3. 配置nginx反向代理

```bash
server {
    listen 80;
    server_name  demo.fastapi.com;
    rewrite ^(.*)$  https://$host$1 permanent;
}

server {
    listen 443 ssl http2;

    server_name  demo.fastapi.com;;

    # ...省略ssl部分...

    charset utf-8;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_pass http://127.0.0.1:8080;
    }

    location ~ /\.(?!well-known).* {
            deny all;
    }

    access_log  /var/log/nginx/access_fastapi-demo.log;
    error_log  /var/log/nginx/error_fastapi-demo.log error;
}
```

4. 启动Gunicorn

```bash
service gunicorn start

# 查看服务状态
service gunicorn status
```

5. 重启

```bash
# 平滑重启 主进程id不会变
service gunicorn reload

# 重启
service gunicorn restart
```

6. 设置开机启动

```bash
systemctl enbale gunicorn
```