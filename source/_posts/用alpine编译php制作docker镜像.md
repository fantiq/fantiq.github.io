---
title: 用alpine编译php制作docker镜像
tags: [docker alpine php]
categories: [docker]
date: 2018-04-17 17:31:36
---
内容主要包括 1. docker原理 2. alpine系统 3. php的编译安装 4. 镜像的制作
<!-- more -->
# 关于docker
docker作为容器化技术，方便运维对目前分布式、集群的部署进行编排 以及 机器的扩容。docker本质并不是一个硬件隔离的系统，而是通过文件系统以及进程等资源的namespace功能实现了表面上的隔离，在宿主机上 docker 容器其实本质是一个进程。容器实现了软件运行环境的隔离，也就是说对底层包的依赖lib与宿主机进行隔离，这样在进行部署或者增加机器的时候不用关心宿主机的环境，宿主机只需要提供内核以及docker服务就好。

linux系统的启动过程是：
```
# https://www.cnblogs.com/codecc/p/boot.html
BIOS		硬件加电自检 查找并加载MBR
MBR			查找BootLoader 加载grub
GRUB		通过grub配置加载系统内核
Kernel		装载驱动 挂载rootfs 执行/sbin/init
Init		OS初始化 执行 runlevel 相关程序
Runlevel 	启动指定级别的服务
```

docker在上面说了，其实是一个线程，所以docker并没有bios、mbr、grub、kernel 这些过程，docker容器使用的是宿主机的内核

docker在使用上其实分为 镜像(image)与容器(container)
镜像相关命令
```
docker images 				列出所有的镜像
docker rmi [image_id] 		删除镜像
docker pull [image_name]	下载镜像
docker build Dockfile		通过脚本构建镜像
docker load
docker save
docker commit
docker push
```
容器相关命令
```
docker ps -a 						列出所有的容器
docker create -it -v -p [image_id]	创建容器
docker run -it -v -p [image_id]		创建并启动容器
docker start [image_id]				启动容器
docker restart [image_id]			重启容器
docker stop [image_id]				停止容器
docker rm [container_id]			删除容器
docker rename [old_name new_name]	修改容器名字
docker exec -it [image_id cmd]		在容器里执行命令 且 交互式的
docker top [image_id]
docker logs [image_id]
docker export [image_id] ???
docker import [image_id] ???
```

综上来看 docker其实在文件上来讲就是一个文件系统，下文要说的嵌入式系统 [alpine](https://www.alpinelinux.org/) 就提供这样的文件系统，我们接下来会使用他的文件系统来制作base image。alpine是一个嵌入式的 所以系统占用空间非常小，docker官方镜像也在使用alpine作为base image，也可以`docker pull alpine`或者docker官方的镜像，我这边拉取的镜像存在网络问题，所以我是自己import文件系统制作的镜像。

通过系统文件制作镜像
```
wget http://dl-cdn.alpinelinux.org/alpine/v3.7/releases/x86_64/alpine-minirootfs-3.7.0-x86_64.tar.gz
gunzip alpine-minirootfs-3.7.0-x86_64.tar.gz
cat alpine-minirootfs-3.7.0-x86_64.tar | docker import - alpine:3.7.0

# 此时执行 `docker images` 就可以看到你创建的镜像
docker iamges

REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
alpine              3.7.0               b05c13e3639e        47 seconds ago      4.139 MB

```
# 关于alpine
alpine官网有对这个系统的简单介绍
>Small. Simple. Secure.
Alpine Linux is a security-oriented, lightweight Linux distribution based on musl libc and busybox.

可以看出alpine与centos相比，主要的不同是使用了不同的底层包的工具链，centos使用的是glibc 而 alpine使用的是 musl libc

alpine除了小外，官网提供的[pkg](https://pkgs.alpinelinux.org/packages)系统也是很全的，软件比较新 比 yum 好用的多，我在编译php的过程中可以使用apk完成所有的第三方包的下载安装，不需要自己手动编译第三方包

apk命令
```
# 常用命令
apk add [pkg_name]
apk del [pkg_name]
apk search [pkg_name]
apk info [pkg_name]
apk update

# 常用参数
--no-cache				不要从本地缓存中读取
--repository [url] 		指定pkg仓库地址 有时候需要下载非stable版(edge) 的时候需要

```
# 在alpine里编译php
在alpine进行php源码的编译需要对应平台的编译工具，需要用到的编译工具如下
```
# 这些是通用的gcc编译工具
make autoconf
# 这些看了下大概的介绍 主要是提供了 dl 这样的工具等
build-base abuild binutils 
```

下载php代码 并解压
```
wget https://github.com/php/php-src/archive/php-7.2.4.tar.gz
tar -zxvf php-7.2.4.tar.gz
```

configure
```
./buildconf --force # 生成configure
# 安装依赖包
apk add \
# 命令行  \
bison readline readline-dev \
# 格式 压缩 \
libxslt libxslt-dev libxml2 libxml2-dev openssl openssl-dev bzip2 bzip2-dev curl curl-dev \
# 图片 \
freetype freetype-dev libpng libpng-dev libwebp libwebp-dev libjpeg-turbo libjpeg-turbo-dev \
libsodium libsodium-dev \

# 加密 这里没有稳定版需要指定edge分支的源
apk add argon2 argon2-dev --repository http://dl-cdn.alpinelinux.org/alpine/edge/main --no-cache 

#configure 这里主要编译了cli php-fpm
./configure \
--prefix=/usr/local/php-7.2.4 \
--disable-cgi \
--enable-fpm \
--with-readline \
--with-bz2 \
--with-zlib \
--enable-zip \
--with-password-argon2 \
--with-sodium \
--with-openssl \
--enable-mbstring \
--with-gd \
--with-webp-dir \
--with-jpeg-dir \
--with-png-dir \
--with-freetype-dir \
--enable-gd-jis-conv \
--enable-exif \
--enable-bcmath \
--with-curl \
--enable-sockets \
--with-xsl \
--with-pdo-mysql=shared,mysqlnd \
--disable-rpath \
--without-pear \

make && make install
```

然后还要安装一些扩展 `redis` `swoole` `mongodb` 
```
wget http://pecl.php.net/get/redis-4.0.0.tgz && tar -zxf redis-4.0.0.tgz
wget http://pecl.php.net/get/swoole-2.1.3.tgz && tar -zxf swoole-2.1.3.tgz
wget http://pecl.php.net/get/mongodb-1.4.2.tgz && tar -zxf mongodb-1.4.2.tgz

/usr/local/php-7.2.4/bin/phpize && ./configure --with-php-config=/usr/local/php-7.2.4/bin/php-config
make && make install
```
# 制作docker镜像
通过上一步我们得到了主要的两个二进制文件 `bin/php` `sbin/php-fpm` 以及 对应的配置文件
在制作镜像的时候只需要把对应的依赖包装上 然后将这两个二进制文件丢进镜像就好了

安装依赖包
```
# 依赖包不需要dev包了，这里不需要编译
apk add bison readline libxslt libxml2 openssl bzip2 curl freetype libpng libwebp libjpeg-turbo libsodium 
# 这里的argon2 包需要dev包
apk add argon2 argon2-dev --repository http://dl-cdn.alpinelinux.org/alpine/edge/main --no-cache 

```