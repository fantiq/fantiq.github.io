---
title: 使用phpstorm进行断点调试
tags: [phpstrom,debug,xdebug]
categories: [工具]
date: 2017-06-21 14:06:40
---

xdebug的断点原理就是，通过php的xdebug对php运行的变量文件等信息进行监听，并通过9000端口以及DBGp协议，将这些信息发送给客户端PHPStrom。
PHPStrom 作为IDE对自己内部设置的断点以及接收到的数据进行安排显示，并再次通过9000端口，做出响应到服务端的xdebug。
<!-- more -->
## 1. php中xdebug配置
安装xdebug扩展并重启服务，这里省略说明，具体罗列解释下xdebug断点相关的配置项

php开启扩展的配置
```
#php.ini 
xdebug.remote_enable=on
xdebug.idekey=PHPSTROM
```

xdebug相关参数使用

|key|val|default|explain|
|--|--|--|--|
|xdebug.remote_enable|on|off|是否开启远程调试|
|xdebug.idekey|PHPSTROM|[HOST_NAME]|这个值会放在cookie里 XDEBUG_SESSION=PHPSTORM 区分不同的客户端|
|xdebug.remote_host|localhost|localhost|与客户端(IDE)交互的地址|
|xdebug.remote_port|9000|9000|与客户端(IDE)交互的端口|
|xdebug.remote_handler|dbgp|dbgp|与客户端(IDE)交互的协议|
|xdebug.remote_mode|req|req|调试模式模式|


## 2. PHPSTORM 的配置

在设置中找到如下路径
```
Setting > Languages & Frameworks > PHP > Debug > DBGp Proxy
```
按照如下的值进行填写

|key|val|
|--|--|
|IDE Key|PHPSTORM|
|Host|localhost|
|Port|9000|


## 3. 触发xdebug以及使用

当PHPSTROM需要监听的时候需要先开启开关 `Run > Start Listening for PHP Debug Connections`

触发xdebug的方法目前有两种，在请求的时候
1. cookie 中增加值 `XDEBUG_SESSION=PHPSTORM` 
2. queryString 上带上一些参数，比如 `http://localhost/index.php?XDEBUG_SESSION_START=13679`

### 3.1 cookie中增加值
可以通过chrome扩展方面实现 cookie值的添加与去除

扩展的[下载地址](https://chrome.google.com/webstore/detail/eadndfjplgieldjbigjakmdgkmoaaaoc)

### 3.2 在PHPStrom 中Debug
PHPStrom中点击 `Run > Debug` ，因为这个debug实例是通过浏览器的，所以要给这个debug添加server并关联

##### 3.2.1 创建server
在设置中找到如下路径
```
Setting > Languages & Frameworks > PHP > Servers
```
新增一个server配置，名字自定义，其他的值是，host port ，重要的是 Debugger，这个选项指定调试的时候使用哪个debug引擎


##### 3.2.2 配置debug
在 `Run > Edit Configurations` 中新增，类型选择 `PHP Web Application`，其中Server中选择上一步配置的，URL为要请求的Url，PHPStrom在请求的时候会自动帮你加上 `XDEBUG_SESSION_START=111222`，因为在创建server的时候我们在debugger选项中选择了 `xdebug`
