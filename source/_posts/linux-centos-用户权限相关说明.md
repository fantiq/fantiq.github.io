---
title: linux centos 用户权限相关说明
tags: []
categories: []
date: 2016-06-12 17:05:40
---
linux上用户管理 以及 相应权限
查看 增加 删除用户 修改密码
用户 用户组 用户默认目录 用户shell路径 等
<!-- more -->
## 用户管理 相关文件
#### 1. 查看系统有哪些用户
```
cat /etc/passwd
```
linux上面的用户都会写在上面这个文件中`/etc/passwd`
每行表示一个用户 不同字段通过 `:` 分开 共七个字段
```
root:x:0:0:root:/root:/bin/bash
username:passwd:User Id:Group Id:comment:home dir:shell
```
|字段id|说明|
|---|---|
|1|用户名|
|2|密码 x表示 真正的密码在 /etc/shadow加密存储|
|3|用户id|
|4|用户组id|
|5|注释|
|6|用户目录|
|7|shell路径|

#### 2. 查看用户密码
密码在另一个文件`/etc/shadow`中 这个文件只对`root`用户可读
每行表示一个用户 不同字段通过 `:` 分开 共八个字段

```
root:
$6$7QTaRT5qUIMoBpxK$ZhAkNM5vlPyTuML6QZ0MmJD.LFQ1zh8DkREida1APOMInXBUOPwQEjTlwPxM.IlaF/AovSjwWW9SBvogoYW9o1:
16903:
0:
99999:
7:::
```
|字段id|说明|
|---|---|
|1|帐号|
|2|密码|
|3|自1970-01-01 密码被修改的天数|
|4|多少天后允许修改密码 0 表示随时可修改|
|5|多少天后 系统强制用户修改成新密码 1 表示永远不能修改|
|6|密码过期前多少天 系统提示用户密码将过期 -1 不会有警告|
|7|密码过期后多少天 系统禁用这个帐号 -1 表示永不禁用|
|8|这个帐号被禁用的天数|


#### 3. 查看用户所属组
```
cat /etc/group
```
用户所属组的信息在 `/etc/group` 这个文件中
```
root:x:0:
group_name:passwd:GID:user1,user2..
```
每行表示一个用户组 不同字段通过 `:` 分开 共4个字段

|字段id|说明|
|---|---|
|1|用户组名称|
|2|用户组密码|
|3|用户组id|
|4|用户列表 用户名之间用`,`隔开|

[参考链接](http://blog.chinaunix.net/uid-26100163-id-3011493.html)


## 用户管理 相关命令
#### 0. 查看用户信息
```
id username
```

#### 1. 添加用户
 相关指标: 用户名 所属组 用户主目录 shell目录 注释
```
adduser -c 测试用户 -d /home/fantasy -g root -s /usr/bin fantasy
```
adduser 可选参数
```
-c 用户描述
-g 所属主用户组 默认新建一个用户名同名的用户组
-G 
-d 用户主目录 默认 /home/用户名
-s 用户shell目录 默认 /bin/bash
```
默认情况下
```
useradd username
```
会创建 用户 `username` 同时创建 `username` 用户组 和 `/home/username` 用户目录
可用 `-g` 指定用户组 可用 `-d` 指定用户目录
#### 2. 修改用户
修改用户资源路径
```
usermod -d /home/new -s /bin/bash -g develop -G root 
```
这里的参数更`useradd`含义相同

修改用户名
```
usermod -l old_name new_name
```

#### 3. 删除用户
删除用户在 `/etc/passwd` `/etc/shadow` `/etc/group` 这几个文件中的相关内容
```
userdel -r username
```
-r 同时删除用户主目录

#### 4. 设置密码
给用户添加密码 或修改用户密码
```
passwd username
> 输入密码

# 删除用户的密码
passwd -d username

# 锁定某个用户 使其不能登录
passwd -l username
```

```
su user_name 切换用户 并定位到用户的主目录
```

#### 5. 创建用户组
```
# 增加一个用户组
groupadd group_name
# 增加一个用户组 并指定GID 这个GID不能是已经存在的GID
# 在不指定GID的情况下 系统根据当前最大GID数加一 作为GID
groupadd -g GID group_name

groupadd grp1
tail -n 3 /etc/group 
# grp1:x:500

groupadd -g 510 grp2
tail -n 3 /etc/group 
# grp1:x:500
# grp2:x:510

groupadd grp2
tail -n 3 /etc/group 
# grp1:x:500
# grp2:x:510
# grp3:x:511 #这里显示的GID是从最大的GID (510) 加一确定的(511)
```
修改用户组
```
groupmod [-g] group_name
# 修改
groupmod -n old_grp_name new_grp_name
```
删除
```
groupdel group_name
```



[参考链接](http://www.cnblogs.com/end/archive/2011/05/25/2057129.html)

## 用户权限

## 目录的 可读(r) 与 可执行(x) 权限的区别

