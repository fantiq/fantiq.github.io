---
title: html5中引入的csp防跨域
tags: [csp,安全,跨域攻击,http]
categories: [运维]
date: 2016-06-07 22:36:25
---

CSP = content security policy 是浏览器的一个安全策略，如同跨域功能
通过相应头可以控制 浏览器的行为 比如：内联样式 内联js 引用外部js 图片等文件
这个是html5 的新特性 ，用来防止 跨站攻击 很有效 但是要高版本浏览器才会支持
[这里可以看到兼容性](http://caniuse.com/#feat=contentsecuritypolicy)

<!-- more -->

举个例子 在`nginx`上的一个请求头的配置
```
add_header 'Content-Security-Policy' "
default-src 'self' http://cdn.bootcss.com;
script-src 'self' 'unsafe-inline' 'unsafe-eval' http://*.kuaiqiangche.com;
style-src 'self' 'unsafe-eval' http://*.bdimg.com;
report-uri http://csp.report.kuaiqiangche.com;
";
```

## CSP响应头
不同浏览器对http头的识别是不一样的
历史原因导致 开发 需要做不同浏览器的兼容
下面列出一些浏览器识别的头
```
# 通用头
Content-Security-Policy
# firefox 响应头
X-Content-Security-Policy
# chrome safari
X-WebKit-CSP
```
## key值的格式
不同的csp指令之间用 `;` 隔开 一条CSP规则是通过
csp指令 值1 值2 .. 的形式 通过空格隔开
```
指令1 值1 值2;指令2 值1 值2;....

default-src 'self' http://baidu.com;
script-src 'self' http://baidu.com;
...
```

## CSP 指令

其实csp的指令主要是 在html中`src`属性 
浏览器会根据这个`src` 的属性 发送请求指定地址
这也是浏览器安全问题出现的根源
所以在csp指令中都是对这个src可以让浏览器发送请求的`白名单`

|指令|例子|描述|
|---|---|---|
|default-src|'self' cdn.example.com|定义所有类型(js img css font ajax iframe 多媒体)可发送请求的白名单|
|script-src|'self' js.example.com|定义js可被浏览器加载的白名单域|
|style-src|'self' css.example.com|定义css可被浏览器加载的白名单域|
|img-src|'self' img.example.com|定义image可被浏览器加载的白名单域|
|connect-src|'self'|定义ajax websocket 可请求交互的白名单|
|font-src|font.example.com|web font 可加载白名单|
|object-src|'self'|定义<object><embad><applet> 这些flash java小程序可被请求加载的白名单|
|media-src|media.example.com|定义<audio><video>等多媒体资可加载白名单|
|frame-src|'self'|定义 iframe 可加载白名单|
|sandbox-src|allow-forms|对资源启用sandbox(类似iframe的sandbox属性?)|
|report-uri|log.kqc.com|定义当请求不被策略允许的时候发送报告到的位置|



## CSP 指令值
|指令值|指令试列|说明|
|---|---|---|
|*|img-src *|允许任何内容|
|'none'|style-src 'none'|不允许任何内容|
|'self'|img-src 'self'|允许相同源(这里的定义如同跨域定义的相同源概念)|
|'unsafe-inline'|script-src 'unsafe-inline'|允许行内资源(如 style属性 onclick时间 inline js css)|
|'unsafe-eval'|script-src 'unsafe-eval'|允许加载动态代码 eval方法|
|https:|img-src https:|允许加载https: 资源|
|data|img-src data|允许加载 data: 协议的资源(如base64的图片)|
|www.example.com|img-src www.example.com|允许加载指定域名的资源|
|*.example.com|img-src *.example.com|允许加载指定域的子域名所有资源|
|https://img.com|img-src https://img.com|允许加载指定域的资源 http协议也要匹配|



在实际项目用常用到的其实并没有这么多
前端用到的资源很多 这个维护量很大 全面使用 CSP策略 给开发也会带来很多不便
我目前在项目中的使用是这样的

```
script-src 'self' 'unsafe-inline' 'unsafe-eval' 
http://*.kuaiqiangche.com http://*.kuaiqiangche.cn 
https://*.kuaiqiangche.com https://*.kuaiqiangche.cn 
....
```


[参考链接](http://drops.wooyun.org/tips/1439)
