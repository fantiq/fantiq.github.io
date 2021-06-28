---
title: linux centos screen的使用总结
tags: [linux,centos,screen]
categories: [运维]
date: 2016-06-07 22:35:15
---
我们知道一些需要一直执行的命令如 `node app.js` `php artisan queue:work --daemon`
如果我们在shell中来执行 当shell关闭这些线程会因此中断。因为他们是通过shell这个进程
来执行的 属于 shell进程的一个线程 ，当我们关闭shell 这个shell的进程也会随之关闭
当然shell这个进程维护的线程也同样会停止，怎样才能让这些命令像service一样不会因为
shell的关闭而停止呢？方法有三
<!-- more -->
1. 做成service
2. 做一个不会停止的shell进程

当然成本最低的是方法2 最容易想到的也是方法2 这里讲解下方法2 的实现 `screen`

## 1 安装
安装软件
```
yum install screen -y
```
## 2 配置文件
无
## 3 启动服务
无
## 4 使用命令总结

创建一个tty
[pid.]tty.host 这个是一个完整的格式
tty 是可以同名的
screen会给你分配一个pid的
如果你有多个同名的tty
screen -r pid 也是可以进去的
```
screen -S tty
```
查看有哪些session
```
screen -ls
```
进入到一个指定的tty
```
screen -r tty
screen -r pid
screen -r pid.tty
```
多用户情况下 一个tty只能一个用户使用
detached 表示离线 可以`screen -r` 进去
attached 表示在线的 只允许单个用户进入
不过你可以强制挤掉别人的登录
```
screen -D -r tty
```
或者这样实现共享
```
screen -x -r tty
```
当想要删除一个screen的时候该怎么操作呢？
首先要 `screen -r session_id ` 进入
其次 通过 `exit` 退出就删除了这个 session

一个screen的tty里面是可以创建多个window的 通过命令
这里的命令都是以 ctrl+a(C-a) 开始的


C-a c # create 新建一个window
C-a n # Next 切换到下一个window
C-a p # Previous 切换到上一个window
C-a 0..9 # 切换到第0...9个window
C-a x # 锁定当前window
C-a d # 离开当前window screen后台运行
通过`exit`退出一个window 当一个tty的window全部退出 这个tty也会相应被删除
C-a w # 列出所有的window


[这里有更详细的解释](http://justdo2008.iteye.com/blog/1888772)

**另外** 如何查看 当前是否在一个screen中？ 看你的shell的title
![图片](2016-06-08_195315.png)
