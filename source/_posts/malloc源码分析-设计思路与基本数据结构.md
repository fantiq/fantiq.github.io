---
title: malloc源码分析-设计思路与基本数据结构
tags: [malloc]
categories: [malloc,c]
date: 2019-05-13 15:15:56
---
malloc 属于应用层的内存管理软件，malloc的实现代码有多个，其中glibc中的实现使用的是 `ptmalloc` 这个是最早的实现了，后面的实现是基于`ptmalloc` 来实现，分析 `ptmalloc` 的源码有助于对malloc算法的理解
<!-- more -->

ptmalloc 支持多线程，每个线程会分配一个arena(arena 数量是有最大值的，最大值与内核数量相关)，arena又根据 heap_info 进行分组，多个 heap_info 组成了 arena
内存的最小单位是 chunk

ptmalloc相当于一个缓存 fastbin smallbin largebin unsortedbin topbin last_remainder



