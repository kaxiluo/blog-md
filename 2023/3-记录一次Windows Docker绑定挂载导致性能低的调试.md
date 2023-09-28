记录一下多年前我使用Windows Docker作为开发环境，运行基于PHP-FPM的项目性能低下的问题。
开发过程中，通过Bind mounts与容器共享源码目录，结果项目访问速度十分慢，特别是遇到源码文件特别多的项目(例如magento2)，速度已经慢得无法忍受。

PHP-FPM(进程管理器)：一种传统的PHP运行模式，运行时不同于常驻内存，每次都会加载源码。

Bind mounts(绑定挂载)：Docker容器中管理数据文件的一种方式，一般适合开发过程中，容器与主机共享源码。

## 探寻问题原因

- 借助工具分析性能

利用XHProf(一个轻量级的分层性能测量分析器PHP扩展) + Xhgui(搭建非侵入式监控平台)，跟踪到了程序大部分的运行时间消耗在 `require_once`，该语句的作用是引入php文件。由此确定源码文件的读取是性能瓶颈。

- 绑定挂载导致I/O性能低

再做个对比测试，把源码复制到容器中再访问，果然速度变快了。由此确定绑定挂载导致源码读取速度变慢。
查询资料得知，Docker的绑定挂载通过SMB共享来实现的，再加上跨操作系统的文件转换，从而I/O性能变得极低。

综上，因为源码在windows主机上，通用绑定挂载共享到Docker容器中，文件的I/O性能低下，项目基于php-fpm，每次运行消耗大量时间加载源码，从而导致项目速度变慢。

## Docker绑定挂载性能测试

分别以`Hyper-V`和`WSL2`作为后端引擎，再结合挂载不同位置和配置，测试读写性能和实际项目表现。

### 准备工作

- 安装WSL2

参见 [安装WSL](https://learn.microsoft.com/zh-cn/windows/wsl/install)

因为我之前安装过WSL1，所以我只需要升级Linux发行版到WSL2

```bash
# 升级内核
wsl --update
# 升级到wsl2 
wsl --set-version Ubuntu-22.04 2
```

- 测试I/O工具

通过`dd`命令测试I/O性能

- 测试项目实际表现

一个干净的`Laravel8`项目，仅输出‘hello world’

### 结果记录

基于机械硬盘，Windows Docker, Linux Container

- **Hyper-V + 源码拷贝到容器中**

```bash
# 运行容器
> docker run --name test001 -it --rm -p 8000:8000 php:7.4-fpm /bin/bash
# 主机中执行命令，复制源码到容器中
> docker cp E:/laravel8 test001:/var/www/html

# 在容器中开始测试
> cd /var/www/html/laravel8/
# 写性能测试
> dd if=/dev/zero of=test.dat bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 0.153936 s, 665 MB/s
# 读性能测试
> dd if=test.dat of=/dev/null bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 0.0692353 s, 1.5 GB/s
```

```bash
# 启动内置Web服务，测试项目访问速度
> php artisan serve --host=0.0.0.0 --port=8000
PHP 7.4.27 Development Server (http://0.0.0.0:8000) started

# 浏览器中访问 http://127.0.0.1:8000
第一次 52 ms
第二次 49 ms
第三次 48 ms

# 安装opcache后，再重启Web服务试试
> docker-php-ext-install opcache
# 再次访问，记录耗时
第一次 115 ms
第二次 11 ms
第三次 11 ms

# 退出并自动销毁容器
> exit
```

- **Hyper-V + Bind mounts**

```bash
# 运行容器
> docker run -it --rm -v E:/laravel8:/var/www/html -p 8000:8000 php:7.4-fpm /bin/bash
# 写性能测试
> dd if=/dev/zero of=test.dat bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 17.4388 s, 5.9 MB/s
# 读性能测试
> dd if=test.dat of=/dev/null bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 0.328262 s, 312 MB/s
```

```bash
# 启动内置Web服务，测试项目访问速度
> php artisan serve --host=0.0.0.0 --port=8000
PHP 7.4.27 Development Server (http://0.0.0.0:8000) started

# 浏览器中访问 http://127.0.0.1:8000
第一次 1.41 s
第二次 656 ms
第三次 616 ms

# 安装opcache后，再重启Web服务试试
> docker-php-ext-install opcache
# 再次访问，记录耗时
第一次 701 ms
第二次 36 ms
第三次 37 ms

# 退出并自动销毁容器
> exit
```

- **Hyper-V + Bind mounts + 一致性选项**

绑定挂载的一致性选项：`consistent`、`delegated`、`cached`。
但是这个选项仅仅适用于Docker for Mac。

*切换Docker引擎 ‘Use the WSL 2 based engine’，重启计算机*

- **WSL2 + Bind mounts**

```bash
# 运行容器
> docker run -it --rm -v E:/laravel8:/var/www/html -p 8000:8000 php:7.4-fpm /bin/bash
# 写性能测试
> dd if=/dev/zero of=test.dat bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 6.20594 s, 16.5 MB/s
# 读性能测试
> dd if=test.dat of=/dev/null bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 5.87995 s, 17.4 MB/s
```

```bash
# 启动内置Web服务，测试项目访问速度
> php artisan serve --host=0.0.0.0 --port=8000
PHP 7.4.27 Development Server (http://0.0.0.0:8000) started

# 浏览器中访问 http://127.0.0.1:8000
第一次 1.73 s
第二次 801 ms
第三次 764 ms

# 安装opcache后，再重启Web服务试试
> docker-php-ext-install opcache
# 再次访问，记录耗时
第一次 1.75 s
第二次 586 ms
第三次 574 ms

# 退出并自动销毁容器
> exit
```

- **WSL2 + Bind mounts + 挂载源目录改为`/mnt/e/`**

```bash
## 进入子系统，以下命令均在子系统执行
# 运行容器
> docker run -it --rm -v /mnt/e/laravel8:/var/www/html -p 8000:8000 php:7.4-fpm /bin/bash
# 写性能测试
> dd if=/dev/zero of=test.dat bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 6.97992 s, 14.7 MB/s
# 读性能测试
> dd if=test.dat of=/dev/null bs=1024 count=100000
100000+0 records in
100000+0 records out
102400000 bytes (102 MB, 98 MiB) copied, 6.67416 s, 15.3 MB/s
```

```bash
# 启动内置Web服务，测试项目访问速度
> php artisan serve --host=0.0.0.0 --port=8000
PHP 7.4.27 Development Server (http://0.0.0.0:8000) started

# 浏览器中访问 http://127.0.0.1:8000
第一次 1.71 s
第二次 761 ms
第三次 760 ms

# 安装opcache后，再重启Web服务试试
> docker-php-ext-install opcache
# 再次访问，记录耗时
第一次 1.72 s
第二次 579 ms
第三次 583 ms

# 退出并自动销毁容器
> exit
```

- **WSL2 + Bind mounts + 挂载源目录改为`//wsl$`**

> WSL2作为后端引擎，Docker Desktop安装了两个专用的内部Linux发行版和，分别用于运行Docker引擎、存储容器和镜像。两者都不能用于一般的开发。

文件资源管理器中输入 `\\wsl$` 查看

```bash
# 尝试把源码放在 \\wsl$\docker-desktop-data\laravel8
# 无法挂载...
```

## 总结

在Windows中，对于源码文件不多的php-fpm项目，选择`Hyper-V + Bind mounts` 再启用`opcache` ，基本可以愉快的开发。

但是遇到源码文件特别多的项目，例如magento2，我还是选择远程开发！

## 选择远程开发

为了方便在Windows编写代码，Linux容器中运行，选择远程开发也是一个很好的方式。如今的IDE对远程开发的支持已经很完美了。

只需在容器中安装ssh服务，映射22端口到主机即可。下面给个例子供大家参考。

**Docekerfile**
```bash
FROM debian:buster

RUN set -xe \
    && apt-get update -y \
    && apt-get install -y --no-install-recommends \
       openssh-server

RUN mkdir -p /var/run/sshd \
  && sed -i "s/#PermitRootLogin prohibit-password/PermitRootLogin yes/" /etc/ssh/sshd_config \
  && echo root:123456 | chpasswd

WORKDIR /workspace

CMD ["/usr/sbin/sshd", "-D"]

EXPOSE 22
```

**构建镜像&运行**
```bash
docker build -t debian-remote-dev-test .

docker run -d -p 2202:22 debian-remote-dev-test
```

**测试SSH连接**
```bash
ssh root@127.0.0.1 -p 2202
...连接成功...
```


记录一下我使用PhpStorm远程开发时踩的坑

- 默认同步选项配置未勾选 '本地删除后删除远程文件'，然后我在本地重命名或删除了文件，结果远程还保留了之前的文件，导致程序出错了。

- 开发过程中，远程生成的一些辅助文件(代理类)，忘记同步到本地，导致本地代码提示不全。

## 参考

[Docker Storage](https://docs.docker.com/storage/)

[Shared Volumes Slow](https://github.com/docker/for-win/issues/188)

[博客 | Docker桌面WSL2最佳实践](https://www.docker.com/blog/docker-desktop-wsl-2-best-practices/)
