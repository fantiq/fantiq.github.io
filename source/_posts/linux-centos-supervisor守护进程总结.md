---
title: linux centos supervisor守护进程总结
tags: [linux,centos,screen]
categories: [运维]
date: 2016-06-07 22:35:30
---

有时候我们通过`screen` 可以让程序不会随着shell的关闭而关闭 ，虽然程序在执行 但是
在各种情况下 会有一些问题导致这个程序停止掉。在liunx上一些著名的开源程序提供一个
守护进程的方法 通过监听代码是否在执行 如果不再执行就自动重启他 从而保证服务的稳定
比如 `php-fpm` 这样的东西 ， 但是当我们开发一个需要一直执行的脚本 如何能有一个守护进程
来保证服务的稳定呢？自己开发守护进程的成本太高 linux有一个提供这个功能的开源程序
`supervisor` 这里就总结下这个软件的使用

<!-- more -->


## 1 安装
安装
```
yum install supervisor -y
```
查看是否安装成功
```
service supervisord status
```
## 2 配置
```
vim /etc/supervisord.conf

# 配置详解
[program:任务名称]
# 守护执行的命令
command=/root/test/shell/test
# 是否自动启动
autostart=true
# 是否自动重启
autorestart=true
startsecs=3
# 是否捕获错误
redirect_stderr=true
# 错误日志
stdout_logfile=/root/test/shellsupervisor.log
# 命令执行的用户 这里设置出错会导致权限问题
user=root
```
## 3 启动
服务端
```
service supervisord start
# 或者
supervisord
```
用户端
```
supervisorctl

supervisor> help # 帮助命令
supervisor> status # 查看任务状态
supervisor> start # 启动任务
supervisor> stop # 停止任务
supervisor> restart # 重启任务
supervisor> reload # 重新加载配置文件 修改配置文件一定要reload后才会生效
```
## 4 使用
比如我们创建一个每10分钟将当前时间写入到一个文件的shell
/root/test/shell/time

```
#!/bin/bash
while true
do
    sleep 10
    echo `date` >> time.log
done

给以执行权限
```
`chmod u+x time`
```
给所有人添加执行权限
chmod +x filename
给所有人添加执行权限
chmod a+x filename
给所属组添加执行权限
chmod g+x filename
给其他人添加执行权限
chmod o+x filename
给文件所属人添加执行权限
chmod u+x filename
```
测试执行命令 
`./time`

加入到supervisor

```
vim /etc/supervisord.conf

[program:super_test]
command=/root/test/shell/time
```
执行客户端
```
# 进入客户端
supervisorctl
# 重新加载配置文件
reload
# 这时查看 这个任务已经启动 我们设置了自动启动
status
```
**注意**
配置里面的command命令要写全路径 suerpvisor有这样一个小[bug](http://www.plope.com/software/collector/215/)

[参考链接](https://fukun.org/archives/07102224.html)
