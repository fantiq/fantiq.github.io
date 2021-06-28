---
title: php并发控制
tags: []
categories: [技术笔记]
date: 2017-04-06 18:12:59
---
这两天遇到一个常见的并发控制的问题，类似抢票问题，当剩下一张票的时候 两个人同时抢这张票 可能会出现多卖的情况
<!-- more -->
在发短信的时候 一般会限制一个手机号发送一次的时间间隔在60s
我们的代码大概会这么写
```
# 获取上次发送的时间
$time = getLastSendTime();
if (time() - $time > 60) {
    sendSms();
    # 更新最后发送时间
    updateLastSendTime(time());
}
else {
    不能发送
}
```
看似严谨的逻辑 但是在并发的情况下 同一时间 两个请求发送过来的话 情况就不一样了
可能第一个请求还没有执行到 `updateLastSendTime` 第二个请求就已经执行到判断语句了
这个时候 程序判断会允许这个请求发送短信的，当请求是成千上万的并发 短信就会灾难性的被刷掉了

同样在上面买票的场景，并发过来的时候，会出现超卖的情况

看下面代码，重现下这个问题 (这里为了简化问题 使用文件代替redis 等持久化存储)
```
<?php
$interval = 60;
if (time() - getLastSendTime() > $interval) {
    $msg = "send sms";
    echo $msg;
    _log($msg);
    updateLastSendTime();
}
else {
    $msg = "please request after $interval s";
    echo $msg;
    _log($msg);
}

/* 更新最后发送时间 */
function updateLastSendTime()
{
    return file_put_contents('time', time());
}
/* 获取最后发送时间 */
function getLastSendTime()
{
    if (!file_exists('time')) {
        updateLastSendTime();
        return time();
    }
    return file_get_contents('time');
}
/* 日志记录 */
function _log($content)
{
    file_put_contents('sendsms.log', date('Y-m-d H:i:s') . " : " . $content . "\n", FILE_APPEND);
}
```

当一个请求触发脚本 我们收到日志
```
2017-04-06 10:57:16 : send sms
```

当我们果断时间再次请求 我们收到日志
```
2017-04-06 10:57:36 : please request after 60 s
```

这个是我们一般情况下的状态，下面我们使用 `ab(apache beanch)` 工具来测试下并发的情况

我们发送5个用户 的并发
```
ab -c 5 -n 5 http://localhost/lock.php
```

这个时候我们得到的日志是这样的
```
2017-04-06 11:04:22 : send sms
2017-04-06 11:04:22 : send sms
2017-04-06 11:04:22 : send sms
2017-04-06 11:04:22 : please request after 60 s
2017-04-06 11:04:22 : please request after 60 s
```

同一时刻发送的五个请求 前三个都成功了 你的逻辑被羞辱了 ==!

**注意** 这里产生的原因是进程问题导致
如果你使用php自带的serv`php -S 127.0.0.1:8081/lock.php` 就不会出现这个问题 
那是因为他是单进程/单线程运行的 所有的请求是进行排队的

但是如果你使用的`nginx` 那个这个问题会很明显，因为nginx有强大的吞吐量(基于epoll模型)
你开的进程越多 问题越明显，我这里开发环境开的是三个nginx进程 正对应了上面的日志

当然线上服务为了更好服务更好的请求 会开更多的进程。


那个如何解决这样的问题呢？
这两天实验了两种方法
1. 队列
2. 文件锁

队列一个目的是为了 代码解耦，这里要把整个逻辑代码搬到队列 不太合适
文件锁可以在控制器层来做，且按需求来做，但是高并发用户量的时候这么做可能会让其他请求用户一直等待
可能等到http请求60秒超时...

这里使用`文件锁`的方法先解决这个问题

```
#给文件上锁
# source 是文件资源句柄 fopen() 的返回
# LOCK有以下几个取值
# LOCK_EX 排他锁 被锁定的文件 在被其他进程上锁的时候 需要阻塞等待 文件被解锁
# LOCK_SH 共享锁
# LOCK_UN 释放锁
# LOCK_NB 非阻塞 仅适用于 linux
bool flock(source, LOCK)
```

下面加入锁的功能

```
lock(function () {
    if (time() - getLastSendTime() > 10) {
        $msg = "send sms";
        echo $msg;
        _log($msg);
        updateLastSendTime();
    }
    else {
        $msg = "please request after 60 s";
        echo $msg;
        _log($msg);
    }
});


function lock($callback)
{
    $fp = fopen('.lock', 'w+');
    if (flock($fp, LOCK_EX)) {
        call_user_func($callback);
    }
    fclose($fp);
}

......
```

这个时候再次 
```
ab -c 5 -n 5 http://localhost/lock.php
```

得到日志：
```
2017-04-06 11:21:32 : send sms
2017-04-06 11:21:32 : please request after 60 s
2017-04-06 11:21:32 : please request after 60 s
2017-04-06 11:21:32 : please request after 60 s
2017-04-06 11:21:32 : please request after 60 s
```

只有第一次执行成功，后面的都失败 判断成功了。
加锁的目的就是让第一个请求先执行完(记录最后一次发送的时间)，在执行第二个请求
这样判断才能在并发的场景生效



## 并发请求 数据库重复插入数据 解决办法
### 1. mysql 唯一索引
### 2. mysql 加锁
MyISAM 表级锁
InnoDB 行级锁

Write 
Read
### 3. redis 加锁 或者文件加锁
### 4. 队列
