---
title: virtualbox 搭建LNMP环境
tags: [virtualbox,lnmp]
categories: [运维]
date: 2016-08-01 10:20:11
thumbnailImage: lnmpa.jpg
thumbnailImagePosition: right

---

lnmp 环境搭建 以及 vagrant box文件打包

<!-- more -->

## 1. virtualbox centos 6.5 安装
[virtualbox 下载地址](https://www.virtualbox.org/wiki/Downloads)
[centos 6.5 x64 下载地址](http://www.centoscn.com/CentosSoft/iso/2013/1205/2196.html)
安装过程 不再详细记录 记得在启动后读取光盘内容的时候可能需要你手动在设置里面加载下
## 2. 安装后的准备
#### 网络设置
```
# 开启ip自动分配
vi /etc/sysconfig/network-scripts/ifcfg-eth0
ONBOOT=yes
# 重启网卡服务
service network restart
# 检查ip
ip a
```
#### 安装wget vim 等常用工具
```
yum install wget
yum install vim
```
#### 更换yum源 epel
```
rpm -Uvh http://mirrors.kernel.org/fedora-epel/6/i386/epel-release-6-8.noarch.rpm
yum repolist
```
## 3. nginx
#### 下载源码 并解压
```
wget https://nginx.org/download/nginx-1.10.1.tar.gz
tar -zxvf nginx-1.10.1.tar.gz
```
#### 安装必备工具
```
yum -y install gcc autoconf automake
yum -y install zlib zlib-devel openssl openssl-devel pcre-devel
```
#### 编译安装
```
cd nginx-1.10.1

./configure \
--prefix=/usr/local/nginx \
--sbin-path=/usr/local/nginx/sbin/nginx \
--conf-path=/usr/local/nginx/conf/nginx.conf \
--error-log-path=/var/log/nginx/error.log \
--pid-path=/var/run/nginx/nginx.pid \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_flv_module \
--with-http_gzip_static_module \
--http-log-path=/var/log/nginx/access.log \
--http-client-body-temp-path=/var/tmp/nginx/client \
--http-proxy-temp-path=/var/tmp/nginx/proxy \
--http-fastcgi-temp-path=/var/tmp/nginx/fcgi \
--with-http_stub_status_module

make && make install
```
#### 启动nginx
```
##### 添加nginx运行用户以及对应用户组
groupadd -r nginx 
useradd -s /sbin/nologin -g nginx -r nginx
##### 启动服务
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
在启动的过程中报错
```
nginx: [emerg] mkdir() "/var/tmp/nginx/client" failed (2: No such file or directory)
```
创建对应的文件即可
```
mkdir -p /var/tmp/nginx/client
```
检查是否启动成功
```
ps -ef | grep nginx
--------------------
root      9422     1  0 13:25 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nginx     9423  9422  0 13:25 ?        00:00:00 nginx: worker process         
```
找到nginx进程即启动成功
## 4. php

#### 下载源码并解压
```
wget http://cn2.php.net/distributions/php-7.0.9.tar.gz
tar -zxvf php-7.0.9.tar.gz
```
#### 安装依赖包
```
yum -y install libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel MySQL pcre-devel curl curl-devel libmcrypt libmcrypt-devel
```
#### 编译安装
```
cd php-7.0.9

./configure \
--prefix=/usr/local/php7 \
--exec-prefix=/usr/local/php7 \
--bindir=/usr/local/php7/bin \
--sbindir=/usr/local/php7/sbin \
--includedir=/usr/local/php7/include \
--libdir=/usr/local/php7/lib/php \
--mandir=/usr/local/php7/php/man \
--with-config-file-path=/usr/local/php7/etc \
--with-mysql-sock=/var/run/mysql/mysql.sock \
--with-mcrypt=/usr/include \
--with-mhash \
--with-openssl \
--with-mysqli=shared,mysqlnd \
--with-pdo-mysql=shared,mysqlnd \
--with-gd \
--with-iconv \
--with-zlib \
--enable-zip \
--enable-inline-optimization \
--disable-debug \
--disable-rpath \
--enable-shared \
--enable-xml \
--enable-bcmath \
--enable-shmop \
--enable-sysvsem \
--enable-mbregex \
--enable-mbstring \
--enable-ftp \
--enable-gd-native-ttf \
--enable-pcntl \
--enable-sockets \
--with-xmlrpc \
--enable-soap \
--without-pear \
--with-gettext \
--enable-session \
--with-curl \
--with-jpeg-dir \
--with-freetype-dir \
--enable-opcache \
--enable-fpm \
--with-fpm-user=nginx \
--with-fpm-group=nginx \
--without-gdbm \
--enable-fileinfo

make && make install
```
#### 添加redis扩展
```
# 下载解压包
wget http://pecl.php.net/get/redis-3.0.0.tgz
tar -zxvf redis-3.0.0.tgz
# 生成configure 文件
cd redis-3.0.0
/usr/local/php7/bin/phpize
./configure --with-php-config=/usr/local/php7/bin/php-config
make && make install
```
[参考链接](http://my.oschina.net/u/1156660/blog/365451)

#### 开启 `opcache` `redis` `pdo_mysql` 必备扩展
```
#file : etc/php.ini

extension=redis.so
extension=pdo_mysql.so
zend_extension=opcache.so
```
[opcache 配置参数参考](http://www.cnblogs.com/HD/p/4554455.html)
#### 启动fpm服务
如果在 `etc` 目录找不到 `php.ini` 文件 可以去源代码里面 `cp` 一份
`php-fpm` 配置文件需要准备
```
/usr/local/php7/sbin/php-fpm
```
#### 检查是否启动
```
ps -ef | grep php-fpm
-----------------------
root     13900     1  0 16:57 ?        00:00:00 php-fpm: master process (/usr/local/php7/etc/php-fpm.conf)
nginx    13901 13900  0 16:57 ?        00:00:00 php-fpm: pool www           
nginx    13902 13900  0 16:57 ?        00:00:00 php-fpm: pool www           
root     13904  1479  0 16:58 pts/0    00:00:00 grep php-fpm

```
#### 将php添加到系统路径
```
ln -s /usr/local/php7/bin/php /usr/local/bin
```
## 5. nginx + php-fpm
首先通过ip访问 如果访问不通 查看防火墙
```
service iptables status
```
如果防火墙开启 请先关闭
```
service iptables stop
```
#### php配置文件开启真实文件路径的支持
```
# file : etc/php.ini
cgi.fix_pathinfo=1
```
#### nginx 配置文件
```
server {
    listen 80;
    service_name localhost;
    # 绑定路径 这个要定义在server下
    root /root/dev;
    index index.html index.htm;
    location / {}
    error_page  500 502 503 504 /50x.html;
    location = /50x.html {
        root html;
    }
    location ~ \.php$ {
        # 代理地址 以及转发 端口
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        # 这个要设置正确 不然 会报 file not found
        # 配合php.ini 的 cgi.fix_pathinfo=1 使用
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```
## 6. mysql

#### 下载解压源码
```
wget http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.14.tar.gz
tar -zxvf mysql-5.7.14.tar.gz
cd mysql-5.7.14
```
#### 安装依赖工具
mysql 5.7 必须的三个依赖
ncurses 字符终端处理库
cmake 跨平台的编译工具 类似(automake)
boost c++ 的一个标准库
bison GNU的一个语法分析器
```
# 可以通过 rpm -q 查看软件包是否安装
yum -y install gcc gcc-c++ ncurses-devel cmake bison
# 下载 boost
wget https://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.tar.gz
tar -zxvf boost_1_59_0.tar.gz
mv boost_1_59_0 /usr/local/boost
# 创建mysql用户 以及用户组
useradd mysql
```
#### 编译安装
```
cmake . \
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
-DMYSQL_DATADIR=/usr/local/mysql/data \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DMYSQL_TCP_PORT=3306 \
-DMYSQL_USER=mysql \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_ARCHIVE_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_MEMORY_STORAGE_ENGINE=1 \
-DDOWNLOAD_BOOST=1 \
-DWITH_BOOST=/usr/local/boost

# 编译出错需要清除缓存
rm -f CMakeCache.txt
make clean
```

>顺便一提: php nginx 都是用纯c写的 mysql用的是c++ 在cmkae mysql的时候
>竟然发现没装gcc-c++ 

编译安装的时候出现如下错误
```
mysql Fatal error: can't close CMakeFiles/innobase.dir/sync/sync0rw.cc.o: No space left on device
```
这是因为mysql需要写文件到 `/tmp` 目录 , 但是在linux上面 `/tmp` 目录是一个固定大小(十几KB)的目录
解决办法就是将系统的临时文件夹换掉(通过修改系统变量来实现)
```
mkdir /home/tmp
# 只在本次登录有效
export TMPDIR=/home/tmp
```

最后换掉还是不行 是因为虚拟机分配的硬盘容量太小
通过增加容量解决编译问题 [如何给virtualbox硬盘扩容]()

#### 启动服务
安装后需要先初始化下mysql
```
# --initialize 初始化mysql  会给root生成默认随机密码
# --initialize-insecure 初始化mysql 但 不会给root生成随机密码
# --user 指定mysql执行的用户
# --basedir mysql的安装路径
# --datadir 指定数据存储路径 这个路径必须是空文件夹 当然也要有读写权限
/usr/local/mysql/bin/mysqld --initialize-insecure --user=mysql --basedir=/usr/local/mysql --datadir=/var/lib/mysql/data
```
将mysql安装成服务
```
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chkconfig --add mysqld
chkconfig mysqld on
service mysqld status
# 启动mysql的方式
service mysqld start
/etc/init.d/mysqld start
```
mysql启动时候默认的配置文件是 `/etc/my.cnf`

```
# 客户端配置
[client]
port=3306
socket=/var/lib/mysql/mysql.sock
#服务端配置
[mysqld]
port=3306
socket=/var/lib/mysql/mysql.sock
# 文件目录
datadir=/var/lib/mysql/data
# 服务启动用户
user=mysql
```

[参考链接](http://www.2cto.com/database/201510/447675.html)

[官方安装参考链接](http://dev.mysql.com/doc/refman/5.7/en/source-installation.html)


## 7. 其他相关软件
node composer git nfs samba supervisor
```
# composer 先将php加入系统路径
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
composer -v

# git
yum -y install git
# redis
yum -y install redis
# nfs
yum -y install nfs-utils
[使用参考链接](http://fantiq.github.io/2016/06/07/linux-centos-%E7%A1%AC%E7%9B%98%E5%85%B1%E4%BA%ABnfs%E4%BD%BF%E7%94%A8%E6%80%BB%E7%BB%93/)
# samba
yum -y install samba
[使用参考链接](http://fantiq.github.io/2016/06/07/linux-centos-samba-%E6%96%87%E4%BB%B6%E5%85%B1%E4%BA%AB/)
# supervisor
yum -y install supervisor
[使用参考链接](http://fantiq.github.io/2016/06/07/linux-centos-supervisor%E5%AE%88%E6%8A%A4%E8%BF%9B%E7%A8%8B%E6%80%BB%E7%BB%93/)
```

## 8. virtualbox 增强工具

#### 挂载增强工具光盘
```
设备 -> 安装增强工具 -> VBoxGuestAdditions.iso(文件在virtualbox 安装目录 这个iso盘会被自动挂载)
mount /dev/cdrom /mnt/cdrom
```
遇到错误提示 
mount: block device /dev/sr0 is write-protected, mounting read-only
是正常的 因为光盘就是 read only 的
可以加参数 避免 错误提醒 : 
```
umount /mnt/cdrom
mount -t auto -o ro /dev/cdrom /mnt/cdrom
```
[mount 命令具体参考](http://blog.chinaunix.net/uid-8810763-id-2453890.html)

#### 安装
```
# 安装依赖
yum -y install bzip2 make automake gcc gcc-c++ kernel-devel kernel-headers
# 更新linux 内核源码 并重启服务器 [参考](https://srobb.net/rhkerndev.html)
yum update
# 执行sh脚本
./VBoxLinuxAdditions.run
```
如果后面有如下错误提示 : 
```
Building the OpenGL support module                         [失败]
```
这个问题可以不解决 不想看到这个提示可以在编译的时候忽略这个错误
```
export MAKE='/usr/bin/gmake -i'
```
[virtualbox 增强包安装 参考](http://my.oschina.net/tashi/blog/190060)

#### 如何使用
先设置虚拟机的共享 `设置>共享文件夹>固定分配` 自动挂载 启动机器 挂载硬盘
```
mount -t vboxsf 共享目录名称 服务器要挂载的目录
```
## 9. 打包成 box文件

#### 1 创建vagrant用户并添加到sudo
```
useradd vagrant
passwd vagrant
vagrant

# 添加行
visudo
vagrant ALL=(ALL) NOPASSWD:ALL
```

sudoer 文件格式
```
user host=(run_as) command
# user 定义用户或者用户组 用户组以 % 开始 如 %vagrant
# host 主机名
# run_as 作为哪个用户执行 不写(run_as) 默认是 root
# command 可执行的命令 这个命令要写全路径 相对路径存在安全问题(用户可以创建同名脚本 以root用户执行脚本 脚本里的命令就可以按照root用户的权限执行了)
# command 前可以通过 NOPASSWD: 来修饰 表示sudo执行命令的时候不用密码
%vagrant ALL=(root) NOPASSWD:/usr/sbin/useradd
# 上面命令定义 vagrant组的用户可以在所有主机以root身份执行 /usr/sbin/useradd 命令且不需要密码
```
[参考链接](http://my.oschina.net/aiguozhe/blog/38706)


```
#### 2 设置ssh
创建文件夹 
```
# 写入 vagrant 中提供的公钥 到用户.ssh 文件夹的 authorized_keys文件中
cd /home/vagrant/.ssh
wget https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -O authorized_keys

chmod 600 /home/vagrant/.ssh/authorized_keys
```
#### 打包box
# 首先通过virtualbox命令查询要打包的虚拟机名称
VBoxManage list vms
# 打包
vagrant package --base VBoxName --output dev.box
# 设置vagrant box存储位置 设置环境变量
VAGRANT_HOME
# 添加box
vagrant add box centos65 /foo/bar/dev.box
# 用box初始化环境 
vagrant init centos56
# 修改配置文件
Vagrantfile
config.ssh.insert_key = false
# 启动
vagrant up
```