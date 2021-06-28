---
title: vagrant的安装与使用
tags: [vagrant,docker]
categories: [运维]
date: 2016-07-26 11:52:04
thumbnailImage: logo.jpg
thumbnailImagePosition: right
autoThumbnailImage: yes
coverImage: cover.jpg
coverCaption: "vagrant"
metaAlignment: center
coverMeta: out
---
作为统一开发环境的工具 vagrant 是一个类似 docker 的工具但是两者的又有不同的使用场景
<!-- more -->
本文所在的环境
```
windows 64位 旗舰版
vagrant 1.8.5
virtualbox 5.0.2
```

## 1. vagrant是什么?
vagrant 是一个方便操作 virtualbox 的工具
实现原理 virtualbox 是开源的 操作对外提供有api vagrant 通过这些api实现对virtual的操作
vagrant 一个方便操作虚拟机的工具 

## 2. 安装虚拟机 vagrant
[下载virtualbox](https://www.virtualbox.org/wiki/Downloads)
[下载vagrant](https://www.vagrantup.com/downloads.html)
vagrant 支持 VMware 虚拟机 但是 VMware 收费 并且 VMware的box文件可下载的也很少 下文只针对 VMware进行讲解
## 3. 下载box镜像

```
# 通过官网别名安装
vagrant box add alias
# 通过下载地址 或者 本地box文件路径安装
vagrant box add [alias] <url | path>
```
镜像地址可以是官方的名称 但是官方的名称就是别名 你不能再通过 --name 指定别名
也可以使 url 系在地址 或者 下载box文件到本地 通过指定box文件路径进行安装 (这两个都是可以定义别名的)

这个命令执行后会在你的用户目录创建一个 `.vagrant.d` 文件夹 里面的`boxes`文件夹存储你安装的box相关文件
包括网络下载的以及本地导入的 都会存储

## 4. 初始化工作目录
```
vagrant init [alias]
```
这时候会在当前目录生成配置文件 `Vagrantfile`
主要配置文件选项

## 5. 启动虚拟机

对应启动虚拟机时候会遇到如下三个问题

#### 1 "rsync" could not be found on your PATH
###### 错误提示
```
"rsync" could not be found on your PATH. Make sure that rsync
is properly installed on your system and available on the PATH.
```

###### 解决方法 ([github issue 参考链接](https://github.com/mitchellh/vagrant/issues/6696))
找到 你下载的 `box` 镜像文件所在路径 一般是 `用户目录\vagrant.d\boxes\镜像名称下的目录\virtualbox\Vagrantfile` 将这段
```
config.vm.synced_folder ".", "/home/vagrant/sync", type: "rsync"
```
修改成
```
config.vm.synced_folder "你要共享的目录", "/home/vagrant/sync", type: "virtualbox"
```
或者将这段注释掉




#### 2 Warning: Authentication failure
###### 错误提示
```
    .......
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
    default: Warning: Authentication failure. Retrying...
Timed out while waiting for the machine to boot. This means that
Vagrant was unable to communicate with the guest machine within
the configured ("config.vm.boot_timeout" value) time period.

If you look above, you should be able to see the error(s) that
Vagrant had when attempting to connect to the machine. These errors
are usually good hints as to what may be wrong.

If you're using a custom box, make sure that networking is properly
working and you're able to connect to the machine. It is a common
problem that networking isn't setup properly in these boxes.
Verify that authentication configurations are also setup properly,
as well.

If the box appears to be booting properly, you may want to increase
the timeout ("config.vm.boot_timeout") value.
```

###### 解决办法 ([github issue 参考链接](https://github.com/mitchellh/vagrant/issues/5186))
a. 配置文件增加
```
config.ssh.insert_key = false
```
b. 通过安装插件(经测试无效)
```
vagrant plugin install vagrant-rekey-ssh
```

#### 3 Vagrant was unable to mount VirtualBox shared folders
###### 错误提示
```
    default: /vagrant => D:/wamp/Vagrant/kqc
Vagrant was unable to mount VirtualBox shared folders. This is usually
because the filesystem "vboxsf" is not available. This filesystem is
made available via the VirtualBox Guest Additions and kernel module.
Please verify that these guest additions are properly installed in the
guest. This is not a bug in Vagrant and is usually caused by a faulty
Vagrant box. For context, the command attemped was:

set -e
mount -t vboxsf -o uid=`id -u vagrant`,gid=`getent group vagrant | cut -d: -f3`
vagrant /vagrant
mount -t vboxsf -o uid=`id -u vagrant`,gid=`id -g vagrant` vagrant /vagrant

The error output from the command was:

mount: unknown filesystem type 'vboxsf'
```
###### 解决办法 ([github issue 参考链接](https://github.com/mitchellh/vagrant/issues/3341))

通过安装插件 `vagrant-vbguest`
```
vagrant plugin install vagrant-vbguest
```

## 6. 常用vagrant命令
box 镜像管理
```
vagrant box add 
vagrant box list
vagrant box remove
vagrant box update
```

plugin 插件操作
```
vagrant plugin list
vagrant plugin install
vagrant plugin uninstall
vagrant plugin update
```

package
```
vagrant package --base [VBoxName] --output [BoxName]
```
机器操作相关
```
# 初始化一个目录
vagrant init
# 启动服务器
vagrant up
# 关闭服务器
vagrant halt
# 重启服务器
vagrant reload
# 查看服务器状态
vagrant status
# 删除服务器
vagrant destroy
# 通过ssh登录服务器 (windows 不可用 请使用xshell 或者 putty类似软件)
vagrant ssh
# 查看服务器登录信息
vagrant ssh-config
# 查看版本
vagrant version
```

## 7. 制作自定义box
#### 环境说明
```
windows 64位 旗舰版
vagrant 1.8.5
virtualbox 5.0.2
```
#### 设置环境变量
vagrant 会将box文件存储在一个文件件 默认存储在 用户目录的 `vagrant.d` 文件夹中
这个文件夹一般位于C盘 由于我C盘空间不足 我重新定以了这个文件夹路径
通过设置环境变量 如下

|Key|Val|
|--|--|
|VAGRANT_HOME|F:/vagrant/|

#### 准备VirtualBox 文件
安装centos系统
安装VirtualBox扩展包
安装需要的软件
....
创建 vagrant 用户 并添加到 sudoer中
```
# 添加用户
useradd vagrant
# 设置密码
passwd vagrant
vagrant
# 添加到 sudoer
visudo
%vagrant ALL=(ALL) NOPASSWD:ALL
```
添加vagrant的公钥到系统
```
wget https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -O /home/vagrant/.ssh/authorized_keys
chmod 600 /home/vagrant/.ssh/authorized_keys
# 关机
halt
```

#### 制作box文件
```
# 使用VirtualBox提供的工具
VBoxManage list vms
# 打包
cd VBox 存储虚拟机文件的位置
vagrant --base [VBoxName 上面命令列出来的名字] --output [box文件名]
```

#### 使用
```
# 添加box
vagrant box add [alias] [box文件路径]
# 初始化
vagrant init [alias]
# 配额制文件 添加配置项
config.ssh.insert_key = false
# 启动
vagrant up
```