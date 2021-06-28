---
title: linux centos 硬盘共享nfs使用总结
tags: [centos,linux,nfs]
categories: [运维]
date: 2016-06-07 22:35:57
---
当我们多个服务器要共享同一块硬盘的时候 比如一些公共代码库 比如公共图片资源文件资源
这时候就需要通过磁盘共享的方式来实现。通过nfs可以满足我们的这个需求 下面深入讲解下
nfs的使用
nfs = network file system
NFS 在文件共享传输的过程 依赖RPC(Remote Procedure Call)协议
<!-- more -->
## 1 安装nfs
```
yum install nfs-utils -y
```
[参考这里](http://www.cnblogs.com/argb/p/3438568.html)
查看是否安装成功
```
rpm -qa | grep nfs
#------- 下面这里表示安装成功 ---------
nfs-utils-1.2.3-70.el6.x86_64
nfs-utils-lib-1.1.5-11.el6.x86_64
```

## 2 文件配置
编辑配置文件
```
vim /etc/exports

#/foo/bar:client_ip(读写权限,同步或异步,)
/root/share 192.168.30.*(rw,root_squash,sync)
```
## 3 开启服务
```
service rpcbind start
service nfs start
```
开启`rpc`服务可以
```
service rpcbind start
# 或者
/etc/init.d/rpcbind start
```
contos 5.x 版本使用的是`portmap`
centos 6.x 版本使用的是`rpcbind`
## 4 权限配置
编辑完配置文件需要重启nfs服务或者 
```
exportfs -rv
```
重新加载权限
## 5 挂载共享硬盘
```
mount -t nfs 192.168.30.52:/root/share /root/test
```
**注意**
挂载成功后需要你查看挂载路径的权限


[参考资料1](http://www.cnblogs.com/mchina/archive/2013/01/03/2840040.html)
[参考资料2](http://www.360doc.com/content/14/0526/19/1123425_381208263.shtml)