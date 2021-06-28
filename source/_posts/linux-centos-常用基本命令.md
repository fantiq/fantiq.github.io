---
title: linux centos 常用基本命令总结
tags: [linux,centos]
categories: [运维]
date: 2016-05-31 19:53:02
thumbnailImage: logo.jpg
thumbnailImagePosition: right

coverImage: cover.jpg
coverCaption: "linux command"

---
linux centos 6.x 常用的一些命令总结
这里不讲基本用法 多讲一些很有用的小技巧

<!-- more -->

## 1 常用的文件操作
```
# change directory 切换目录
cd dir

# 列出当前文件加下的所有文件
ls

# 更细的列出文件夹下的所有文件 -a 可以列出隐藏文件
ll -a

# 拷贝文件 -r 拷贝一个目录
cp -r source dist

# 拷贝远程文件 远程文件地址 username@ip:/foo/bar
scp source dist

# 移动文件 或者修改文件名称
# -f 覆盖不询问 -n 覆盖询问 -b 文件在覆盖前做备份 (备份文件名末尾加 ~ )
mv source dist

# remove 删除文件 -r 删除文件夹 -f 强制删除不提示
rm -rf

# 创建一个快捷方式 软连接
ln -s source dist 
```

## 2 文件包的安装
```
# 不用确认直接安装
yum install pkgname -y
# 查询已经安装的包
yum list pkgname
# yum搜索一个包
yum search pkgname
# 删除安装的包
yum remove pkgname

# 查询安装的包
rpm -qa | grep pkgname
# 查询安装包的详细信息
rpm -ql | grep pkgname

```


## 3 用户相关
```
# 当前登录用户
whoami
id
# 切换登录用户
su username
# 修改用户密码
passwd username
```
## 4 文件相关

创建一个空文件

```
# 创建一个空文件
touch filename
# 覆盖写入内容到一个文件 没有文件则创建
echo "" > filename
# 追加写入内容到文件
echo "" >> filename
```
查看文件
```
# 查看全部文件
cat filepath

# 分页浏览文件

# 
# more 翻页 快捷键
#         上翻        下翻
# 一行      -         回车
# 一页      b         f 空格
# q 退出
# 
# 
more filepath

# 
# less 翻页 快捷键
#         上翻        下翻
# 一行     y k         j 回车
# 半页      u           d
# 一页      b           f 空格
# q 退出
# 同样可以用vi的一同快捷键
# j k 上下移动 gg 开头 G 结尾
# ctrl+f 下翻一页  ctrl+b 上翻一页 f:front  b:back
# ctrl+u 下翻半页  ctrl+d 上翻半页 u:up     d:dowm

less filepath

# 显示文件部分内容 并且内容是实时更新的

# 显示文件开头 默认前10行
# -c 显示前K字节的内容
# -n 显示前n行的内容 默认10行
# -q 
# -v 显示文件名称
head filepath
# 显示文件结尾 默认后10行
# -c 显示前K字节的内容
# -n 显示前n行的内容 默认10行
# -q 
# -f 
# -F 
# -s 文件刷新的频率
tail filepath

# 
sed
awk
```

## 5 压缩与解压缩

打包压缩过程
打包的过程可以直接调用压缩程序处理文件

#### 打包
```
tar
    -c 新建文档
    -x 解压缩
    -r 向压缩文档结尾追加文件
    -u 更新原压缩文件中的文件
    # 压缩 解压 都能用到的参数 但只能用其中一个参数
    -z gzip
    -j bzip2
    -Z compress
    -v 显示进程
    -O 将文件解开到标准输出流

    # 必须作为最后一个参数
    -f 文档名称
```
举例 解压一个文件 `nginx.tar.gz`

```shell
# x 解压缩
# z 解压算法 gzip
# v 显示解压进程
# f 指定文档
tar -xzvf nginx.tar.gz
```

举例 打包一个文件 `nginx`

```shell
# c 创建打包文件
# z 选择文件压缩算法
# v 显示进程
# f 指定压缩后的文件名
#
# tar -czvf 打包后的文件名 要打包的文件或文件夹
tar -czvf my.tar.gz nginx
```


#### 压缩

##### linux系统

|文件类型|后缀|压缩|解压缩|
|--|--|--|--|
|gzip|.gz|gzip file|gzip -d / gunzip|
|bzip2|.bz2|bzip2|bunzip2|
|compress|.Z|compress|uncompress|


##### windows系统

|文件类型|后缀|压缩|解压缩|
|--|--|--|--|
|zip|.zip|zip|unzip|
|rar|.rar|rar|unrar|

## 6 其他常用

```
# ssh登录
ssh username@ip

# 查看当前文件夹下面文件大小
du -ha

# 查看服务
service servname [start|stop|restart]

# 查看当前系统运行的服务
ps -ef | grep servname

# 查找一个文件
# -name 查找文件名 -type 查找文件类型
find path [option] name


# 硬盘挂载
mount -t nfs ip:/foo/bar dist
# 取消挂载
umount dist 

# 显示当前操作系统名称
uname -a

```
[linux命令大全 参考链接](http://man.linuxde.net/)
