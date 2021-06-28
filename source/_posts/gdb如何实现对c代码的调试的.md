---
title: gdb如何实现对c代码的调试的
tags: [c,gdb,调试]
categories: [c]
date: 2019-03-11 16:57:07
---
dbg 是一个调试工具，这里记录下他是如何实现对源代码的调试的。一般情况下，在IDE中配置好调试配置，然后在源代码中打断点就可以进行调试了。需要的配置是 1. 指定调试工具 2. 指定运行的可执行文件。那问题是，调试软件`gdb` 是如何将 可执行文件与源代码文件对应起来的呢？


## gdb 原理
gcc编译器的过程是 `源代码 -> 预处理代码 -> 汇编代码 -> 目标码` 在这个过程中从目标码到源代码的对应关系是被记录下来的，通过`gcc -g` 参数会将这些对应关系记录下来。

我们在运行程序的时候，可以通过在源代码中指定的断点信息 (哪个文件的哪一行) ，通过编译过程产生的对应关系，找到可执行文件的执行处。
使用 `gcc -g` 编译的可执行文件 中会包含源代码信息，可以通过gdb读取到这些信息 以php为例进行演示如下
```
>gdb
file ./php
l 1,20 # 查看源代码1到20行
b 10 # 在第10行进行断点
run # 执行调试
```

下一步要做的就是干预可执行文件的执行过程了，这个就需要依赖`gdb` 来实现。`gdb` 的主要实现是通过系统函数 `ptrace` 对执行的进程进行侵入式的干预。

`long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data)`
参数说明

|类型|参数|作用|
|--|--|--|
|enum __ptrace_request|request [PTRACE_TRACEME PTRACE_ATTACH PTRACE_DETACH]|执行的动作|
|pid_t|pid|要跟踪的进程id|
|void *|addr|要监控的内存地址|
|void *|data|存放读写数据|

gdb通过这个方法对执行的线程进行捕获，截获发送到被追踪程序信号。

gdb实现了两个形式的断点：
1. 通过gdb 启动进程， gdb作为父进程对监控进程进行监控
2. gdb通过 `ptrace(PTRACE_ATTACH, 被监控程序的)` 使自己成为被监控程序的父进程

## mac上无法使用gdb的原因
在mac上使用`gdb`的时候会报错
```
Unable to find Mach task port for process-id xxxx: (os/kern) failure (0x5).
```

这是由于 `mac` 系统的保护机制，禁止一个进程对另一个进程有强烈的干预操作，他推荐自己家的 `lldb` 来替代  `gdb` 。
要使用`gdb` 就要对其进行证书授权，方式请参照参考链接

遇到 `During startup program terminated with signal SIG113, Real-time event xxx `  这个错误的话，同样是系统的保护机制 需要关闭下 系统的 SIP 保护机制，配置如下
```
#file .gdbinit
set startup-with-shell off
```


## vscode调试php内核代码
在调试php内核代码的时候 一直使用 `CLion` 进行，中间遇到很多问题，换 `vscode` 方便多了，调试c代码需要一个 插件 `C/C++`就可以了。`vscode` 主要需要配置的地方是两个地方 一个是 `task` 的配置(配置文件 `task.json`)， 一个是 `launch` 的配置(配置文件 `launch.json`)，其中 `task.json` 跟代码的调试并无关，他相当于IDE提供的一个自定义命令 可以在IDE中执行自定义命令，跟调试相关的是 `launch.json` 这个文件
```
{
    "request": "launch", // launch attach 
    "program": "可执行文件路径",
    "args": [参数1, 参数2, ...], // 对应可执行文件的参数
}
```

其中 "request" 可选的值有 `launch` 和 `attach` 

`launch`
通过调试程序启动可执行程序，以父进程的形式对被跟踪程序进行监控

`attach`
这是通过将调试进程通过附加的形式，成为正在执行的进程的父进程来实现监听，所以这里是需要给出一个被监听进程的id的，配置应该增加
```
{
    "request": "attach",
    "processId": "被调试的进程id"
    "program": "",
    "args": [],
}
```



> 参考链接
> [GDB调试原理——ptrace系统调用](https://www.cnblogs.com/xsln/p/ptrace.html)
> [将源程序和汇编指令映射起来](https://www.kancloud.cn/itfanr/i-100-gdb-tips/81888)
> [MAC上使用gdb](https://blog.csdn.net/github_33873969/article/details/78511733)
> [VSCode Debugging](https://code.visualstudio.com/docs/editor/debugging)
