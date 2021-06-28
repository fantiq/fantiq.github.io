---
title: JAVA中HashMap死循环形成过程的分析
tags: []
categories: []
date: 2020-05-17 20:23:28
---
在JAVA的之前版本，java的hashmap是`拉链法`的实现，为了防止拉链过长导致哈希查询退化成链表的遍历查询，哈希表会进行扩容，并对元素进行重排。
这个情况就是在哈希表重排的时候有并发请求，导致重排进行了两次，从而链表变成了循环链表，这样当读操作读取到这个循环链表则会进入死循环，导致CPU占用率飙升。为了避免这种情况，在创建hashmap的时候使用带锁的hashmap  `ConcurrentHashMap`。
<!-- more -->
在新的JDK中HashMap的实现略有不同，当链表长度超过一定的值时候，会将链表转成红黑树的结构以提高查找效率。

## 哈希表的拉链形式以及扩容
通过hash计算将key转换成一个范围内的数字，然后用一个数组存储这个kv的指针，这里为了简便将key假设为数字 使用mod计算作为hash计算。当hash冲突的时候通过链表的形式将新的kv增加到链表的头部(增加到头部是因为数组存储的是链表的开始，在头部插入可以避免对链表的遍历可以直接进行插入操作)。

#### 拉链
比如将如下kv`(5, C) (7, B) (3, A)`写入到hashmap，map尺寸为宽度为2的数组 `[2]int hashtable`

对3进行mod计算 `5 % 2 = 1` 则 hashtab[1] = {kv: (5, C), next: null}
对7进行mod计算 `7 % 2 = 1` 则 hashtab[1] = {kv: (7, B), next: &kv(5)} {kv: (5, C), next: null}
对5进行mod计算 `3 % 2 = 1` 则 hashtab[1] = {kv: (3, A), next: &kv(7)} {kv: (7, B), next: &kv(5)} {kv: (5, C), next: null}

最终形成的结构如下: 

```
0 null
1 (3, A)-> (7, B)-> (5, C)-> null
```

#### 扩容

将上面hashmap进行扩容，从容量2扩大到容量4，需要对hahsmap中的数据进行重新hash计算并拷贝数据到新的内存中，遍历老的哈希表数据:

对3进行mod计算 `3 % 4 = 3` 则 hashtab[3] = {kv: (3, A), next: null}
对7进行mod计算 `7 % 4 = 3` 则 hashtab[3] = {kv: (7, B), next: &kv(3)} {kv: (3, A), next: null}
对5进行mod计算 `5 % 4 = 1` 则 hashtab[1] = {kv: (5, B), next: null}

最终形成的结构如下:

```
0 null
1 (5, C)-> null
2 null
3 (7, B)-> (3, A)-> null
```


## HashMap扩容死循环的形成

同样，在JDK中的HashMap实现的数据结构也是拉链形式，扩容时候的步骤如下:

1. 读取旧哈希表的数据
2. 计算kvpair中k的hash值
3. 查找对应hash值的链表
4. 将kvpair插入到链表表头

#### 扩容算法

这里使用伪代码表示插入的形式

```
for e = oldHashTable {
    // 遍历老的hashtable数据
    do {
        next = e.next
        // 重新计算hash值
        h = hash(e.data.key)
        // 将 e 插入表头
        hashtable[h] = e
        e.next = hashtable[h]
        // 遍历
        e = next
    } while (e != null)
}
```

死循环形成的过程是在遍历老table且将数据插入到新table中的过程，将上面代码精简，只关注这个关键过程 
```
e : 遍历到的元素
head : 链表头指针

do {
    // step-1 记录链表下一个元素
    next = e.next
    // step-2 插入表头
    e.next = head
    head = e
    // step-3 循环
    e = next
} while (e != null)
```

#### 并发形成死循环

通过上一步的扩容算法进行推导，为了简便，这里只关注其中的一个链表，因为这个死循环的本质就是单向链表在扩容调整后变成了环形。将上面例子的数据也进行简化，只保留key值和下一个元素的地址 则我们关系的元素只有两个了 `A(7, null)` `B(3, null)` ，数组存储的表头地址设为 `head`

旧哈希表的数据结构如下:`head = B` `A(7, null)` `B(3, A)`

假设有两个线程同时执行到上面代码中的步骤 `step-1` 此时两个线程中的值如下 `e = B` `next = A`。然后让`线程-1`先执行，直到整个表的扩容调整完成，此时的数据结构如下

`head = A` `A.next = B` `B.next = null`

然后`线程-2`继续执行，对其进行推导求值。

##### 第 1 次循环

初始值:

|var|val|
|---|---|
|e|B|
|next|A|
|head|null|
|A.next|B|
|B.next|null|

执行代码:

|步骤|代码|已知变量|结果|
|---|---|---|---|
|1|-|-|-|
|2|e.next = head|head = null , e = B|B.next = null|
|3|head=e|e = B|head = B|
|4|e=next|next = A|e = A|
|5|e != null|e = A|true|

##### 第 2 次循环

上次循环得到的初始值:

|var|val|
|---|---|
|e|A|
|next|A|
|head|B|
|A.next|B|
|B.next|null|

执行代码:

|步骤|代码|已知变量|结果|
|---|---|---|---|
|1|next = e.next|e = A, A.next = B|next = B|
|2|e.next = head|head = B, e = A|A.next = B|
|3|head=e|e = A|head = A|
|4|e=next|next = B|e = B|
|5|e != null|e = B|true|

##### 第 3 次循环

上次循环得到的初始值:

|var|val|
|---|---|
|e|B|
|next|B|
|head|A|
|A.next|B|
|B.next|null|

执行代码:

|步骤|代码|已知变量|结果|
|---|---|---|---|
|1|next = e.next|e = B, B.next = null|next = null|
|2|e.next = head|head = A, e = B|B.next = A|
|3|head=e|e = B|head = B|
|4|e=next|next = null|e = null|
|5|e != null|e = null|false|


##### 循环停止

此时停止了循环，然后看值的结果如下：

|var|val|
|---|---|
|e|null|
|next|null|
|head|B|
|A.next|B|
|B.next|A|

查看此时数据 A B 的 next 指针会发现，已经形成了循环链表 `A.next = B` `B.next = A`。此时当数据读取到这个链表则会形成死循环的情况。



参考链接

> [疫苗：JAVA HASHMAP的死循环](https://coolshell.cn/articles/9606.html)