---
title: '[精]LNMP开发环境搭建'
tags: []
categories: []
date: 2017-01-01 21:22:08
---
vagrant与docker
vagrant 是一个对虚拟机的管理工具，方便搭建、移植自己的linux开发环境
docker 是一种容器技术，每个服务(php-fpm mysql redis) 适合做成一个docker容器，方便本地模拟分布式开发环境换，保持开发环境与生产环境的高度一致

[vagrant制作与使用]()
[docker的制作与使用]()
<!-- more -->

搭建本地开发环境

## 1. 系统安装(centos6.8)
#### 1.1 最小化安装centos系统
#### 1.2 调整网络为自动分配(DHCP)
```
# 编辑网卡配置文件
vi /etc/sysconfig/network-scripts/ifcfg-eth0
# 修改此项
ONBOOT=no
# 修改为
ONBOOT=yes
# 重启网卡
service network restart
# 可以ping同域名则 网络修改成功
ping www.baidu.com
```

#### 1.3 分配网络为NAT [virtualbox网络类型区别]()
修改`virtualbox : 设置 -> 网络` 选择NAT
选择`端口转发`并添加一个映射规则
```
# 查看宿主机ip(eth0)
ip a 
```
添加规则如下

|名称|协议|主机ip|主机端口|子系统ip|子系统端口|
|--|--|--|--|--|--|
|ssh|TCP|127.0.0.1|22|10.0.2.15|22|

>通过TCP协议将宿主机的 127.0.0.1 22端口的请求转发到 10.0.2.15的22端口。此时可以通过ssh工具连接虚拟机了

## 2. 更换yum源为epel
```
rpm -Uvh http://mirrors.kernel.org/fedora-epel/6/i386/epel-release-6-8.noarch.rpm
yum repolist
```

## 3. 安装增强工具 为vagrant做准备

安装增强工具前需要先安装依赖包 并更新更新 yum源 并重启
```
# 安装依赖包
yum -y install bzip2 make automake gcc gcc-c++ kernel-devel kernel-headers
# 更新yum包
yum update
# 重启
reboot
```
安装增强工具
```
#挂载增强工具盘
设备-> 安装增强工具
#挂载硬盘
mkdir /mnt/cdrom
mount -t auto -o ro /dev/cdrom /mnt/cdrom
# 执行
cd /mnt/cdrom
./VBoxLinuxAdditions.run
```
如遇到错误提示
```
Could not find the X.Org or XFree86 Window System, skipping
```
属于正常，因为我们没有安装GUI

## 4. 基础服务安装
创建下载源代码的目录 安装下载工具`wget`
```
yum install -y wget
cd ~
mkdir src
cd src
```
#### 4.1 nginx
下载并解压
```
wget https://nginx.org/download/nginx-1.10.1.tar.gz
tar -zxvf nginx-1.10.1.tar.gz
```
安装依赖包
```
yum -y install gcc autoconf automake zlib zlib-devel openssl openssl-devel pcre-devel
```
编译安装
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
启动服务
```
# 添加nginx运行用户以及对应用户组
groupadd -r nginx 
useradd -s /sbin/nologin -g nginx -r nginx
# 启动服务
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```
启动过程中遇到错误
```
nginx: [emerg] mkdir() "/var/tmp/nginx/client" failed (2: No such file or directory)
# 创建对应的目录
mkdir -p /var/tmp/nginx/client
```
#### 4.2 node
下载并解压
```
# 下载文件
wget https://nodejs.org/dist/v6.9.2/node-v6.9.2-linux-x64.tar.xz
# 下载解压工具
yum -y install xz
# 解压
xz -d node-v6.9.2-linux-x64.tar.xz
tag -xvf node-v6.9.2-linux-x64.tar.xz
```
安装(这里下载的是编译好的文件 可以直接使用)
```
# 移动文件
mv node-v6.9.2-linux-x64 /usr/local/node
# 创建软链接
ln -s /usr/local/node/bin/node /usr/local/bin/node
ln -s /usr/local/node/bin/npm  /usr/local/bin/npm
# 查看版本
node -v
npm -v
```
#### 4.3 mysql
下载并解压 
```
wget http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.14.tar.gz
tar -zxvf mysql-5.7.14.tar.gz
```
安装依赖包
```
yum -y install gcc gcc-c++ ncurses-devel cmake bison
# 下载boost
wget https://sourceforge.net/projects/boost/files/boost/1.59.0/boost_1_59_0.tar.gz
tar -zxvf boost_1_59_0.tar.gz
mv boost_1_59_0 /usr/local/boost
```
编译安装
```
cd mysql-5.7.14

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

make && make install
# 编译错误 需要清理make文件
rm -f CMakeCache.txt
make clean
```
创建mysql执行用户
```
groupadd -r mysql
useradd -s /sbin/nologin -g mysql -r mysql
```
初始化mysql
```
# 创建数据库文件存储目录
mkdir -p /var/lib/mysql
# 赋予权限
chown -R mysql:mysql /var/lib/mysql
# 初始化配置
/usr/local/mysql/bin/mysqld \
--initialize-insecure \
--user=mysql \
--basedir=/usr/local/mysql \
--datadir=/var/lib/mysql/data
# --initialize 初始化mysql  会给root生成默认随机密码
# --initialize-insecure 初始化mysql 但 不会给root生成随机密码
# --user 指定mysql执行的用户
# --basedir mysql的安装路径
# --datadir 指定数据存储路径 这个路径必须是空文件夹 当然也要有读写权限
```
启动mysql
1 查找 /etc/my.cnf 是否存在 不存在的话 拷贝一份
```
cp /usr/local/mysql/support-files/my-default.cnf /etc/my.cnf
```
2 修改配置文件
```
[client]
# 配置客户端sock文件 不然 mysql命令报错
# ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)
socket=/var/lib/mysql/mysql.sock
[mysqld]
# 数据库文件存储位置 这个一定要填写正确 且 有读写权限不然会报错 
# Starting MySQL........ ERROR! The server quit without updating PID file (/var/lib/mysql/develop.pid).
datadir=/var/lib/mysql/data
socket=/var/lib/mysql/mysql.sock
# mysql服务执行的用户
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```
3 启动服务
```
/usr/local/mysql/support-files/mysql.serve start
```
4 将mysql添加到系统服务 且 开机启动
```
# 拷贝文件
cp /usr/local/mysql/support-files/mysql.serve /etc/init.d/mysqld
# 查看服务状态
service mysqld status

# 设置为开机启动
chkconfig --add mysqld
chkconfig mysqld on
```

#### 4.4 redis
下载并解压
```
wget http://download.redis.io/releases/redis-3.2.6.tar.gz
tar -zxvf redis-3.2.6.tar.gz
```
编译安装
```
cd redis-3.2.6
make PREFIX=/usr/local/redis install
```
启动服务
```
# 拷贝配置文件
cp redis.conf /usr/local/redis/
# 修改配置文件 redis为后台进程
# 找到
daemonize no
# 修改为
daemonize yes
# 启动服务
/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
```
#### 4.5 mongodb
下载解压
```
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-3.4.1.tgz
tar -zxvf mongodb-linux-x86_64-3.4.1.tgz
# 由于下载的是二进制文件 可以直接使用
cp -r mongodb-linux-x86_64-3.4.1 /usr/local/mongodb
```
启动服务
```
# --dbpath 指定文件存储的目录
/usr/local/mongodb/bin/mongod --dbpath
# 连接客户端
/usr/local/mongodb/bin/mongo
>show dbs
>use admin
>show collections
>db.collection.find()
```
#### 4.6 php
下载解压
```
wget http://cn2.php.net/distributions/php-7.0.9.tar.gz
tar -zxvf php-7.0.9.tar.gz
```
安装依赖包
```
yum -y install 
openssl openssl-devel \		# 支持 stl
curl curl-devel \			# curl网络请求库
libjpeg libjpeg-devel \		# jpeg图片
libpng libpng-devel \		# png 图片
freetype freetype-devel \	# freetype 字体
libxml2 libxml2-devel \		# xml格式解析
libxslt libxslt-devel \		# 
bzip2 bzip2-devel \			# bzip2 压缩库
pcre pcre-devel \			# pcre正则库

```
编译安装
[核心配置选项列表](http://php.net/manual/zh/configure.about.php)

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
配置相关的修改
```
# 添加链接
ln -s /usr/local/php7/bin/php /usr/local/bin
# php配置文件(从源代码中拷贝)
cp php.ini-development /usr/local/php7/etc/php.ini
# php-fpm 配置文件
cp /usr/local/php7/etc/php-fpm.conf.default /usr/local/php7/etc/php-fpm.conf
cp /usr/local/php7/etc/php-fpm.d/www.conf.default /usr/local/php7/etc/php-fpm.d/www.conf
# 修改配置文件
vi /usr/local/php7/etc/php-fpm.d/www.conf
;listen = 127.0.0.1:9000
listen = /dev/shm/php-fpm.sock
# 下面两个选项的注释一定要开启 不然nginx连接fpm的sock会报权限错误
listen.owner = nginx
listen.group = nginx

# 启动服务
/usr/local/php7/sbin/php-fpm
# 查看是否开启成功
ps -ef | grep php-fpm
# 这个时候查看生成的 php-fpm.sock 的创建者与所属组都是 nginx
ll /dev/shm
```


#### 4.7 php-ext
php扩展安装
```
# redis下载解压
wget http://pecl.php.net/get/redis-3.1.0.tgz
tar -zxvf redis-3.1.0.tgz
# mongodb下载解压
wget http://pecl.php.net/get/mongodb-1.2.2.tgz
tar -zxvf mongodb-1.2.2.tgz
# php提供的扩展安装工具
# 在扩展源码根目录下执行
# 生成configure文件
/usr/local/php7/bin/phpize
./configure --with-php-config=/usr/local/php7/bin/php-config
# 编译安装
make && make install
```
配置文件开启扩展
```
vi /usr/local/php7/etc/php.ini
# 添加如下选项
extension=redis.so
extension=mongodb.so
extension=pdo_mysql.so
zend_extension=opcache.so
# 查看扩展
php -m
```
composer 安装
```
# 下载
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
# 验证
composer -v
```
## 5. 常用工具安装
#### 5.1 supervisor
命令行执行脚本的守护进程
```
# 安装
yum -y install supervisor
# 查看是否安装成功
service supervsord status
```
配置文件说明
```
vi /etc/supervisord.conf
# 配置详解
[program:任务名称]
# 守护执行的命令
command=/root/test/shell/test
# 命令执行的用户 这里设置出错会导致权限问题
user=root
# 是否自动启动
autostart=true
# 是否自动重启
autorestart=true
startsecs=3
# 是否捕获错误
redirect_stderr=true
# 错误日志
stdout_logfile=/root/test/shellsupervisor.log
```
基本常用命令
```
# 帮助
supervisorctl help
# 查看状态
supervisorctl status
# 修改配置文件需要重新加载
supervisorctl reload
# 重启 启动 停止
supervisorctl start/restart/stop
```

[参考连接]()
#### 5.2 nfs-utils
网络磁盘共享 图片文件会用到
```
# 安装
yum -y install nfs-utils
# 查看安装情况
rpm -qa | grep nfs
# --- 看到这些包表示安装成功---
nfs-utils-lib-1.1.5-11.el6.x86_64
nfs-utils-1.2.3-70.el6_8.2.x86_64
# 服务端编辑权限
vi /etc/exports
# /path client_ip (读写权限,访问角色,同步还是异步)
/data/images 192.168.0.* (rw,root_squash,async)
# 权限编辑后需要重新加载生效
exportfs -rv
# 开启服务 (客户端 服务端同时需要开启服务)
# contos 5.x 版本使用的是portmap
# centos 6.x 版本使用的是rpcbind
service rpcbind start
service nfs start
# 客户端挂载硬盘
mount -t nfs remote_ip:/path /local/data
```

#### 5.3 samba
目录共享系统
```
# 安装
yum -y install samba
# 查看安装情况
service smb status
```
配置文件详解
```
vi /etc/samba/smb.conf
# 修改以下选项 否则每次访问需要登录
#security = user
security = share

# 添加一个共享目录
[共享的名称]
comment=注释描述
path=/root/share # 共享的文件夹路径
browseable=yes # 是否有浏览权限
guest ok=yes # 访客是否可访问
public=yes  # 是否允许上传
writable=yes # 是否有可写权限

# 重启服务
service smb restart
```

#### 5.4 docker-io
docker安装
```
yum -y install docker-io
service docker status
```
[docker常用命令]()
#### 5.5 git
```
yum -y install git
```
[git命令详解]()

## 6. vim 安装与配置
#### 6.1 vim源码下载解压
```
wget ftp://ftp.vim.org/pub/vim/unix/vim-7.4.tar.bz2
tar -jxvf vim-7.4.tar.bz2
```
#### 6.2 安装依赖包
```
yum install -y lua lua-devel python python-devel perl perl-devel perl-ExtUtils-Embed ruby ruby-devel
# 编译安装 lua-jit
wget http://luajit.org/download/LuaJIT-2.0.4.tar.gz
tar zxf LuaJIT-2.0.4.tar.gz
cd LuaJIT-2.0.4
# 这里安装默认路径 不然编译vim的时候找不到文件
make
make install
```
#### 6.3 vim源码 编译安装
```
./configure \
--prefix=/usr/local/vim7.4 \
--disable-selinux \
--enable-luainterp=yes \
--enable-perlinterp \
--enable-rubyinterp \
--enable-pythoninterp \
--with-python-config-dir=/usr/lib64/python2.6/config \
--enable-fail-if-missing \
--enable-cscope \
--enable-gui=auto \
--enable-gtk2-check \
--enable-gnome-check \
--with-features=huge \
--with-luajit \
--with-x 

make && make install
```

使用 vim 以及错误解决

```
ln -s /usr/local/vim7.4/bin/vim /usr/local/bin/
# 这个时候使用vim报错
# vim: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
# 通过生成软连接 解决
ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/
# 检查插件支持
vim 命令行模式输入
:version
```

#### 6.4 插件 配置文件安装
```
# 拷贝 .vimrc 配置文件
# 创建目录
mkdir -p ~/.vim/bundle
mkdir -p ~/.vim/colors
```

安装包管理器 Bundle
```
# 下载bundle
git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
# 安装插件
:BundleInstall
```

安装插件 Command-T
```
# Command-T 安装 找到下载的源码
cd ~/.vim/bundle/command-t/ruby/command-t
ruby extconf.rb
make
```
安装插件 Ctags
```
wget https://nchc.dl.sourceforge.net/project/ctags/ctags/5.8/ctags-5.8.tar.gz
tar zxvf ctags-5.8.tar.gz
cd ctags-5.8
./configure --prefix=/usr/local/ctags
make && make install

ln -s /usr/local/ctags/bin/ctags /usr/local/bin/ctags
ctags --version
```


## 7. 制作vagrant
#### 7.1 添加sudo用户
```
# 添加用户
useradd vagrat
passwd vagrant
>password: vagrant
#### 7.2  添加vagrant用户到sudoer
visudo
# 增加行
vagrant ALL=(ALL) NOPASSWD:ALL
```
sudo 文件格式解释
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
#### 7.3 创建vagrant登录授权key
```
su vagrant
mkdir -p /home/vagrant/.ssh
cd /home/vagrant/.ssh
wget https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -O authorized_keys
chmod 600 /home/vagrant/.ssh/authorized_keys
```
#### 7.4 vagrant 打包
打包虚拟机

```
# 查看虚拟机
VBoxManage list vms
# vagrant 打包
vagrant package --base VBoxName --output develop.box
# 设置vagrant存储位置 设置系统常量
VAGRANT_HOME=/d/storage/vagrant
```

使用 vagrant 包

```
vagrant box add centos-dev develop.box
vagrant init centos-dev 
#修改Vagrantfile
# 使用帐号密码登录
config.ssh.username="vagrant"
config.ssh.password="vagrant"
# 使用key登录
config.ssh.private_key_path=""


# 启动
vagrant up
```


另外在开发过程中我们常常会修改配置文件
这个时候可以将配置文件放在映射的目录
然后通过软连接(ln -s) 将文件映射到服务读取的配置文件
