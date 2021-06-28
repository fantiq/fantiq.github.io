---
title: Linux进程基本用法
tags: [进程,Linux]
categories: [Linux]
date: 2019-06-25 18:55:55
---
关于进程涉及到的知识点还是很多的，需要全方位理解操作系统才能更好的理解进程。在初步使用了进程相关的glibc函数之后带来了更多的问题需要思考：1. 进程的创建过程fork 主要是对进程结构 `struct task_struct` 的拷贝，这个结构体中包括 进程调度信息、文件系统、文件读写流、虚拟内存等信息 2. 进程与线程的区别，他们共享系统分配的虚拟内存，具体共享的是虚拟内存的那些部分 3. Linux中代码被编译成机器指令以二进制的文件格式存储在硬盘，这个格式叫做elf 不仅仅有机器指令 还是有头信息、符号表等信息节 4. 多进程在操作系统中是如何被调度的 算法是怎样的 5. 用户态到内核态的转换 依赖的是中断 需要搞清楚中断的实现原理 6. 进程可以接收外部信号 这个外部信号本质又是什么。 搞清楚了这些问题，才能搞清楚进程的本质，同时这些内容也是独立的知识体系需要单独来讲了，这里先写下进程的表面形式。
<!-- more -->
## 1. 什么是进程
操作系统介于硬件与程序之间，一方面给程序提供一个统一的操作硬件的接口，另一方面保护硬件不被应用程序破坏。为了实现多任务系统，操作系统实现的多任务功能，这每个任务就是一个进程。操作系统会给每个进程创建一个独立的抽象硬件的数据空间，多任务的实现其实就是CPU每次执行一部分的进程机器指令，这可以通过中断系统来控制CPU的执行行为，需要先执行哪个进程的问题就是进程调度，操作系统会实现。**进程是被内核调度的最小单元**，线程也是可以被内核调度的，这是因为线程在内核态其实也是进程的形式。

## 1. 进程创建
#### 1.1 fork函数
进程的创建可以使用内核方法 `fork` 或者使用 glibc 的 `fork` 方法创建，fork方法会返回两次 当返回的 pid 为0时说明代码是在子进程中执行，当返回的 pid 不为0的时候说明代码是在父进程中执行。
fork函数

|k|v|
|:-|--|
|签名|pid_t fork()|
|头文件|unistd.h sys/types.h|
|参数|无|
|返回值|返回类型 pid_t 定义在头文件 sys/types.h，函数返值 若<0 则创建失败<br/>如果实在主进程执行 返回值为子进程pid 如果是在子进程中执行 返回值为0|
|说明|创建子进程|

fork函数返回的情况比较特殊，函数只会返回一次，但是fork函数从表象上看却好像返回了两次，其实不然，fork如其他函数一样返回一次，只是在调用fork的过程中内核创建了新的子进程，创建之后 父子进程的代码执行点是一样的，这时候进程调度系统会调度执行 父子两个进程各自的代码，最终父子进程的fork函数都会执行完成 也都会返回一个值，这样总共返回的是两个值。
fork在创建子进程的时候会调用 `copy_process` 方法从父进程中复制进程需要的信息到子进程中，父进程在 fork 方法中会得到 子进程的 pid 这个值会放在 寄存器 `eax` 最终通过函数调用栈返回，子进程也存在一个 返回值 `p->tss.eax`，但是在 `copy_process` 的过程中这个值被直接赋值为 `0` 了，这也是为什么在 调度到父进程的时候fork返回值是子进程pid 调度到子进程的时候fork的返回值是0 的原因了。

fork使用实例
```
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();
    if (pid < 0) perror("create process error");
    if (pid == 0) printf("child process pid=%d\n", getpid());
    // 这里的条件还是要写 因为这个条件在父子进程中都会执行
    if (pid > 0) printf("parent process pid=%d,create child process pid=%d\n", getpid(), pid);
    return 0;
}

-------- result --------
parent process pid=24751,create child process pid=24752
child process pid=24752
```

#### 1.2 进程在内存中的分布
对于进程的创建，在刚刚创建完成的时候 父子进程是完全一样的，这个创建过程就是简单的复制过程，但是这两个进程对于系统是平等的，系统会公平调度这两个进程。
进程作为一个系统调度执行的基本单元，在启动的时候就会加载到内存中，linux流行的二进制文件格式是 `elf` 这种格式也定义了内存的布局。

|section|descriptor|
|--|--|
|.stack|用来存储函数多集调用时候的 临时变量、函数参数|
|.head|程序中动态分配(malloc)的内存区域|
|.bss|未初始化的全局变量|
|.data|初始化的全局变量|
|.text|程序编译后的机器指令|

这些是一个程序运行需要的数据，进程在创建的时候其本质就是复制了这些信息，elf具体的格式需要单独分析。

#### 1.3 进程间关系
进程的创建都是由父进程创建的，所以进程的信息中存在 进程id pid 、父进程id ppid。在操作系统执行之初就创建了一个进程 `idle` 其进程id = 0 ，`idle` 进程创建了进程 `init` pid=1 这个就是用户空间中所有进程的父进程，每个进程都是由其父进程创建，所以每个进程都有父进程。
进程组 pgid，每个进程都有一个进程组的属性，这个属性值是从父进程继承复制过来的，这个值其实是一个进程的pid，这个进程可能已经不存在，系统这里使用进程组这个概念的作用是为了方便对进程的管理，没有分组就只能一个一个进程的处理。
会话 sid，会话主要涉及到shell场景，会话id也是从父进程复制过来的值，会话id同样是一个进程的pid，会话id 不会随着进程退出而消失的，会话的目的是为了方便管理shell中执行的进程(作业任务)。会话进程被称作Leader进程，就是进程id等于会话id的进程，只有这个进程才能操作 tty 对应的IO流。可以通过函数 `setsid()` 修改当前进程的会话id，函数调用成功 当前进程的会话id 等于 当前进程的进程组id 等于 当前进程id，也就是说 `sid = pgid = pid`

进程间的关系代码示例
```
int main() {
    pid_t pid1 = fork();
    if (pid1 < 0) perror("create process failed 1");
    if (pid1 == 0) {
        pid_t pid2 = fork();
        if (pid2 < 0) perror("create process failed 2");
        if (pid2 == 0) {
            // 孙子进程
            printf("p2 %d,%d,%d,%d\n", getpid(), getppid(), getpgid(getpid()), getsid(getpid()));
            exit(0);
        }
        // 子进程
        sleep(1);
        printf("p1 %d,%d,%d,%d\n", getpid(), getppid(), getpgid(getpid()), getsid(getpid()));
        exit(0);
    }
    // 父进程
    sleep(2);
    printf("p0 %d,%d,%d,%d\n", getpid(), getppid(), getpgid(getpid()), getsid(getpid()));
    return 0;
}

-------- result --------
p2 58165,58164,58163,1
p1 58164,58163,58163,1
p0 58163,58994,58163,1
```
上面代码使用了sleep阻塞进程的目的是为了展示进程间的父子关系，如果没有阻塞等待，子进程可能会成为孤儿进程 其父进程id为1，当然正常是需要函数 `wait` 来处理的，具体的wait用法放在后面来看。

通过函数 `setsid` 修改会话id
```
int main () {
    pid_t pid = fork();
    if (pid < 0) perror("create process failed 1");
    if (pid == 0) {
        // 子进程调用 setsid 修改会话id 同时进程组id也会随之改变
        setsid();
        printf("p1 %d,%d,%d,%d\n", getpid(), getppid(), getpgid(getpid()), getsid(getpid()));
        exit(0);
    }
    sleep(1);
    printf("p0 %d,%d,%d,%d\n", getpid(), getppid(), getpgid(getpid()), getsid(getpid()));
    return 0;
}
-------- result --------
p1 58358,58357,58358,58358
p0 58357,58994,58357,1
```

#### 1.4 父子进程监控
父进程创建子进程后，父子进程作为单独的CPU调度单元运行，两者的执行顺序是独立的，但是父子进程有存在进程父子关系，如果父进程先执行完退出，子进程挂载到 init进程还好，如果子进程先执行完 父进程没执行完毕 则子进程会成为`僵尸进程`（进程状态 Z ）, 其实此时的子进程资源已经被回收 但是还有一些统计信息存在进程中，需要等待父进程结束的时候才会被系统回收处理这些僵尸进程。僵尸进程是进程的正常状态，是一个已经退出的进程。如果这种状态进程多的话 会占用一些系统资源 比如进程号，虽让僵尸进程不会被进程调度，但是过多的僵尸进会导致进程表太大 程影响调度程序的便利过程。

父进程需要负责回收子进程，处理函数可以使用 `wait`

|k|v|
|--|--|
|签名|pid_t wait(int * )|
|头文件|sys/wait.h|
|参数|可选 子进程退出的状态值，不需要的话 传 NULL|
|返回值|失败返回 -1 成功返回子进程的pid|
|说明|没有子进程 立刻返回，存在子进程 阻塞等待子进程退出|

更灵活的处理子进程退出函数 `waitpid`

|k|v|
|--|--|
|签名|pid_t waitpid(pid_t, int * , int)|
|头文件|sys/wait.h|
|参数1|需要监听的进程id|
|参数2|子进程退出的状态 如果不关系可设置为 NULL|
|参数3|控制参数 不需要刻意设置为0|
|返回值|失败返回 -1 成功返回子进程的pid|
|说明|更丰富的功能 将停子进程|

参数1

|值|说明|
|--|--|
|pid>0|监听进程 进程号为pid 无论是否为当前进程子进程|
|pid=-1|监听当前进程下的任一子进程 功能如同 wait|
|pid=0|监听 与当前进程在一个进程组 的任何进程|
|pid<-1|监听进程组为pid绝对值的任一子进程|

参数2

|||
|||
|||

WIFEXITED(status)   如果子进程正常结束，它就返回真；否则返回假。
WEXITSTATUS(status) 如果WIFEXITED(status)为真，则可以用该宏取得子进程exit()返回的结束代码。
WIFSIGNALED(status) 如果子进程因为一个未捕获的信号而终止，它就返回真；否则返回假。
WTERMSIG(status)    如果WIFSIGNALED(status)为真，则可以用该宏获得导致子进程终止的信号代码。
WIFSTOPPED(status)  如果当前子进程被暂停了，则返回真；否则返回假。
WSTOPSIG(status)    如果WIFSTOPPED(status)为真，则可以使用该宏获得导致子进程暂停的信号代码。


参数3

|值|说明|
|--|--|
|WNOHANG|不阻塞直接返回 如果子进程没有退出返回 0|
|WCONTINUED|？？？|
|WUNTRACED|阻塞等待？？？|


`wait` `waitpid` 的实现方式并不是 基于信号 `SIGCHLD` ，信号是异步通知的，`wait waitpid`函数默认则是阻塞执行的 应该是轮询实现的吧。

## 2. 进程信号
进程在执行的过程中是可以与外界交互的，外界可以给进程发信号，这个发信号的实现原理依赖的是 CPU中断机制，常用的终止进程命令 `kill -9 <pid>` 就是一个给进程发送信号的过程。CPU中集成有处理中断的硬件，发送中断信号其实是发送给CPU，CPU每执行完指令后就会读取中断设备中是否有中断需要处理，发送的信号对应有一个信号值，这个值对应的是IDT(Interrupt Descriptor Table) 表的索引，他的值是命令存储的内存地址，CPU通过这个信号值查找对应的指令存储的内存地址，然后读取指令执行。
#### 2.1 信号的发送与处理
###### 2.1.1 通过 kill 发送信号
发送信号可以使用函数 `kill` 也对应 shell命令中的 `kill`

|k|v|
|--|--|
|签名|int kill(pid_t, int)|
|头文件|signal.h|
|参数1|进程id|
|参数2|信号索引|
|返回值|成功返回 0 失败返回 -1|
|说明|发送信号给进程|

**返回值**
返回值错误码

|错误|值|说明|
|--|--|--|
|EPERM|1|权限不足|
|EINVAL|22|参数信号 不存在|
|ESRCH|3|参数 进程id或进程组 不存在|

**参数进程id**

|值|说明|
|--|--|
|pid > 0|将信号发送给进程pid|
|pid = -1|将信号发送给除init进程之外的所有进程|
|pid = 0|将信号发送给跟当前进程属于一个进程组的所有进程|
|pid < -1|将信号发送给 进程组为 pid 绝对值的所有进程|

**信号**
对于信号的值 有些信号存在多个值 是因为在不同的架构中 对信号定义的值不同，三个值用逗号隔开，第一个值是 alpha 和 sparc架构使用的，第二个是 x86 arm架构使用，第三个值是mips架构使用。

进程状态相关的信号

|signal|value|desc|
|--|--|--|
|SIGTSTP|18,20,24|暂停进程 这个信号由terminal发出 按键Ctrl-z|
|SIGSTOP|17,19,23|暂停进程 通过其他方式发送 这个信号不可被进程捕获处理 |
|SIGCONT|19,18,25|继续进程执行|

中止进程的四个信号

|signal|value|desc|
|--|--|--|
|SIGINT|2|terminal发送中断 对应按键 Ctrl-c|
|SIGQUIT|3|termian 发送信号 停止进程 对应按键 Ctrl-\ 与SIGINT 的不同是 这个信号在内核的默认信号处理中会产生 core dump|
|SIGTERM|15|停止进程 可以被捕获 这是与 SIGKILL 的区别|
|SIGKILL|9|停止进程 不可被捕获 直接停止进程|

SIGINT SIGQUIT 对应的是 terminal 停止进程 信号都可以被捕获，SIGQUIT 会生成 `core dump` 文件
SIGTERM SIGKILL 不针对 terminal SIGTERM 可以被捕获 SIGKILL 不能被捕获

生成`core dump` 需要设置限制条件 `ulimit -c unlimited` 也可以设置其他值 单位 blocks , 若 设置为 `ulimit -c 0` 则不会生成 `core dumo` 文件。
启动进程 通过 `Ctrl-\` 停止进程 当前文件夹下就会生成 `core dump` 文件 `core.<pid>` ，这个文件的生成是内核的 SIGQUIT 信号处理程序生成的。

其他用到的信号

|signal|value|desc|
|--|--|--|
|SIGUSR1|30,10,16|用户自定义信号|
|SIGUSR2|31,12,17|用户自定义信号|
|SIGABRT|6|当调用函数 abort() 触发的信号|
|SIGALRM|14|当调用函 alarm() 时钟定时时间到的时候触发的信号|
|SIGCHLD|20,17,18|子进程退出的时候触发的信号|
|SIGHUP|1|terminal 退出的时候触发的信号|

###### 2.1.2 通过 signal 注册对应信号处理程序
信号处理函数 signal

|k|v|
|--|--|
|签名|void (*handler)(int) signal(int, void (*handler)(int))|
|头文件|signal.h|
|参数1|信号值|
|参数2|信号处理函数内存地址，SIG_IGN 忽略信号， SIG_DFL 使用默认的处理程序|
|返回值|成功返回 处理函数地址 失败返回 -1|
|说明|注册信号处理函数|

现在常用的signal处理程序注册函数是 `sigaction`，signal存在一定的问题：???



```
void handle(int signo) {
    printf("get sig nr %d\n", signo);
    exit(0);
}
int main() {
    printf("parent pid=%d\n", getpid());
    pid_t pid = fork();
    if (pid == 0) {
        printf("child pid=%d\n", getpid());
        while (1);
    }
    // 注册子进程退出信号的处理函数
    signal(SIGCHLD, handle);
    // 发送信号给指定进程
    getchar();
    // 终止子进程
    kill(pid, SIGTERM);
    return 0;
}
-------- result --------
parent pid=63154
child pid=63155
>\n
get sig nr 20
```

## 3. 进程状态
R 运行 被调度
T 暂停
S 睡眠
Z 僵尸



> TODO
> 中断的实现原理 键盘输入输出
> 计算机的启动过程
> 内存
> 文件
> 进程
> 进程调度算法
> 进程与线程
> IO
> 终端执行shell命令的过程
> unix二进制可执行文件格式 elf
> 








