---
title: linux centos samba 文件共享
tags: [linux,centos,samba]
categories: [运维]
date: 2016-06-07 14:01:52
---

系统 linux centos
软件 samba

<!-- more -->

## 1 安装samba

安装软件
```
yum install samba -y
```
检查samba服务是否已经启动
```
service smb status
```
启动服务
```
service smb start
```
或者
```
/etc/init.d/smb start
/etc/init.d/smb restart # 重启服务

```

##2 配置文件
```
vim /etc/samba/smb.conf
```
修改访问方式 不然需要你登录 才能访问
```
将
security = user
修改成
security = share
```
添加一个共享目录
```
[共享的名称]
comment=注释描述
path=/root/share # 共享的文件夹路径
public=yes  # 是否运行上传
browseable=yes # 是否有浏览权限
writable=yes # 是否有可写权限
guest ok=yes # 访客是否可访问
```
**注意：** 
1. 修改配置文件后要重启smbd服务 这个配置文件是给smb服务端读的
2. 无法访问也可能是防火问题 `service iptables status` 查看下是否已经关闭或者`iptables -L`看你的ip是否有访问服务器的权限
3. 权限问题 首先普及下权限的问题
x 1 执行
w 2 写
r 4 读
这几个权限可以任意组合 chmod 来修改文件权限 
当我们访问一个目录的时候 执行x 与读取r是有区别的
读取r 允许列出目录里面的列表(ll) 但是没有读取目录里内容的权限
执行x 无法读取目录里面的列表 但是可以读取目录里面的内容
也就是说 /foo/bar 这个目录你需要在bar目录里面写文件
可以给 bar 777 但是 foo目录你要给x权限 不然 进不去的
[目录权限可以参考这里](http://blog.csdn.net/caomiao2006/article/details/21701791)



##3 重启samba服务
```
service smb restart
/etc/init.d/smb restart
```
##4 配置文件详解

[这里是配置的一个详细解释](http://yuanbin.blog.51cto.com/363003/115761/)
```

#======================= Global Settings =====================================
    
[global]
    # 指定所属的域
    workgroup = MYGROUP
    server string = Samba Server Version %v
    
    netbios name = MYSERVER
    
    interfaces = lo eth0 192.168.12.2/24 192.168.13.2/24 
    # 允许哪些ip访问
    hosts allow = 127. 192.168.12. 192.168.13.
    
# --------------------------- Logging Options -----------------------------
    # 日志文件存储的位置
    log file = /var/log/samba/log.%m
    # 限制单个日志文件的大小 单位 KB
    max log size = 50
    
# ----------------------- Standalone Server Options ------------------------
    # 用户访问samba共享的验证方式
    # user    需要帐号密码登录才能访问 帐号密码的提供以及验证由samba来做
    # share   不需要验证帐号可直接访问
    # server  用过服务器来进行验证
    # domain  通过验证服务器(主域控制器PDC)来进行验证
    security = share
    # 用户后台
    # smbpasswd samba自己的用户密码管理工具 文件形式
    # tdbsam    使用数据库的形式管理帐号密码
    # dapsam    LDAP服务实现帐号密码的管理
    passdb backend = tdbsam


# ----------------------- Domain Members Options ------------------------
    security = domain
    passdb backend = tdbsam
    realm = MY_REALM
    password server = <NT-Server-Name>

# ----------------------- Domain Controller Options ------------------------
;   security = user
;   passdb backend = tdbsam
;   domain master = yes 
;   domain logons = yes
;   logon script = %m.bat
;   logon script = %u.bat
;   logon path = \\%L\Profiles\%u
;   logon path =          
    
;   add user script = /usr/sbin/useradd "%u" -n -g users
;   add group script = /usr/sbin/groupadd "%g"
;   add machine script = /usr/sbin/useradd -n -c "Workstation (%u)" -M -d /nohome -s /bin/false "%u"
;   delete user script = /usr/sbin/userdel "%u"
;   delete user from group script = /usr/sbin/userdel "%u" "%g"
;   delete group script = /usr/sbin/groupdel "%g"
    
    
# ----------------------- Browser Control Options ----------------------------
;   local master = no
;   os level = 33
;   preferred master = yes
    
#----------------------------- Name Resolution -------------------------------
;   wins support = yes
;   wins server = w.x.y.z
;   wins proxy = yes
;   dns proxy = yes
    
# --------------------------- Printing Options -----------------------------
    load printers = yes
    cups options = raw
;   printcap name = /etc/printcap
    #obtain list of printers automatically on SystemV
;   printcap name = lpstat
;   printing = cups

# --------------------------- Filesystem Options ---------------------------
;   map archive = no
;   map hidden = no
;   map read only = no
;   map system = no
;   store dos attributes = yes


#============================ Share Definitions ==============================
# 这几个是一些特殊共享
[homes]
    comment = Home Directories
    browseable = no
    writable = yes
;   valid users = %S
;   valid users = MYDOMAIN\%S
    
[printers]
    comment = All Printers
    path = /var/spool/samba
    browseable = no
    guest ok = no
    writable = no
    printable = yes
    
# Un-comment the following and create the netlogon directory for Domain Logons
;   [netlogon]
;   comment = Network Logon Service
;   path = /var/lib/samba/netlogon
;   guest ok = yes
;   writable = no
;   share modes = no
    
    
# Un-comment the following to provide a specific roving profile share
# the default is to use the user's home directory
;   [Profiles]
;   path = /var/lib/samba/profiles
;   browseable = no
;   guest ok = yes
    
    
# A publicly accessible directory, but read only, except for people in
# the "staff" group
;   [public]
;   comment = Public Stuff
;   path = /home/samba
;   public = yes
;   writable = yes
;   printable = no
;   write list = +staff
[com]
comment = 测试samba共享目录
path = /root/share
public = yes        # 是否允许guest帐号访问
browseable = yes    # 是否允许浏览
writable = yes      # 是否可写
guest ok = yes      # 同义词 public

```
