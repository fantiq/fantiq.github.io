---
title: malloc源码分析-初始化与arena的创建
tags: [malloc,glibc,ptmalloc,c,内存]
categories: [内存]
date: 2019-05-13 15:16:09
---
glibc中的malloc方法其实是映射到 `__libc_malloc` 的，其通过ASM的符号表的形式来实现
```
strong_alias (__libc_malloc, __malloc) strong_alias (__libc_malloc, malloc)
define strong_alias(original, alias)
  .globl C_SYMBOL_NAME (alias) ASM_LINE_SEP
  .set C_SYMBOL_NAME (alias),C_SYMBOL_NAME (original)
```
<!-- more -->
## 1. malloc 整体流程
1
以下是祛除一些辅助功能后的代码
```
void * __libc_malloc (size_t bytes) {
    mstate ar_ptr;
    void *victim;

    // 1.1 原子的形式调用钩子方法 __malloc_hook 用以初始化 ptmalloc
    void *(*hook) (size_t, const void *) = atomic_forced_read (__malloc_hook);
    if (__builtin_expect (hook != NULL, 0)) return (*hook)(bytes, RETURN_ADDRESS (0));

    // 1.2 获取 或 创建 arena 并且会将arena 加锁(mutex 互斥锁)处理
    arena_get (ar_ptr, bytes);

    // ptmalloc 核心逻辑
    victim = _int_malloc (ar_ptr, bytes);
    /* Retry with another arena only if we were able to find a usable arena before.  */

    // 如果 ar_ptr 指定的 arena不为空 也就是说 arena_get 获取arena是成功的 但是 _int_malloc 方法返回的 victim 却为空 这里会重新获取一遍arena arena_get_retry
    if (!victim && ar_ptr != NULL) {
        LIBC_PROBE (memory_malloc_retry, 1, bytes);
        ar_ptr = arena_get_retry (ar_ptr, bytes);
        victim = _int_malloc (ar_ptr, bytes);
    }

    // 解除arena的互斥锁
    if (ar_ptr != NULL) __libc_lock_unlock (ar_ptr->mutex);

    assert (!victim || chunk_is_mmapped (mem2chunk (victim)) || ar_ptr == arena_for_chunk (mem2chunk (victim)));
    // 返回内存地址给用户
    return victim;
}
```
`__libc_malloc` 整体流程分如下几步：
1. 通过钩子函数 `__malloc_hook` 先初始化 `ptmalloc` 且只会初始化一次
2. 通过 arena_get 获取当前线程的 arena 如果不存在当前线程的arena则创建一个arena返回 且 这一步还给arena加上互斥锁
3. 调用核心逻辑 _int_malloc 查询chunk 
4. 如果arena获取成功 上一步却没找到chunk 则通过arena_get_retry 尝试再次分配
5. 查询成功解锁arena 返回地址给用户

这里需要展开的函数有 `__malloc_hook` `arena_get` `_int_malloc` 其中`_int_malloc` 作为核心函数 代码量比较大 会放在下一章进行详细分析，这里主要分析 `__malloc_hook` `__malloc_hook`



## 2. ` __malloc_hook` 初始化ptmalloc

#### 2.1 malloc初始化调用逻辑
malloc初始化调用的钩子方法 定义如下：
```
void *weak_variable (*__malloc_hook) (size_t __size, const void *) = malloc_hook_ini;
```
这里通过 `weak_variable` 进行修饰，为了方便用户对这个钩子方法的替换，这个钩子方法指定到了 函数 malloc_hook_ini，源码如下：
```
static void * malloc_hook_ini (size_t sz, const void *caller) {
    // 清空malloc钩子方法的绑定
    __malloc_hook = NULL;
    // 调用初始化方法
    ptmalloc_init ();
    // 从新走 __libc_malloc 方法
    return __libc_malloc (sz);
}
```
当函数执行进来则会将钩子函数的变量设置为 NULL 也是为了保证这个初始化函数只会被调用一次，初始化的主要代码还是在函数 `ptmalloc_init` ，初始化完毕还是会调用函数 `__libc_malloc`

#### 2.2 ptmalloc_init

下面是初始化的代码，这里主要为了看逻辑过程 删除了一些配置相关的代码
```
static __thread mstate thread_arena attribute_tls_model_ie;
int __malloc_initialized = -1;
static void ptmalloc_init (void) {
    // 如果执行过或者正在执行中这个方法则直接返回
    if (__malloc_initialized >= 0) return;
    __malloc_initialized = 0;

    #ifdef SHARED
        /* In case this libc copy is in a non-default namespace, never use brk. Likewise if dlopened from statically linked program.  */
        Dl_info di;
        struct link_map *l;
        if (_dl_open_hook != NULL || (_dl_addr (ptmalloc_init, &di, &l, NULL) != 0 && l->l_ns != LM_ID_BASE)) __morecore = __failing_morecore;
    #endif
    // thread_arena 记录当前线程的arena指针地址 ptmalloc_init 中指定的是主线程的arena
    thread_arena = &main_arena;
    // 这里对主线程的 arena 进行初始化
    malloc_init_state (&main_arena);

    // 这里省略了一些配置项设置的代码

    // 初始化执行完毕 将变量 __malloc_initialized 赋值为1
    __malloc_initialized = 1;
}
```
结合上面的逻辑可以看出来，初始化的线程一定是主线程，所以这里的当前线程就是 main_arena  `thread_arena = &main_arena`，然后会通过函数 `malloc_init_state` 初始化一些值。
此方法开始的时候设置 `__malloc_initialized=0` 结束的时候设置 `__malloc_initialized=1` 。主要的初始化代码在函数 `malloc_init_state`
需要注意下其中变量 thread_arena 的定义 `static __thread mstate thread_arena attribute_tls_model_ie;` 变量通过关键词 `__thread` 进行修饰，也就是说这个全局变量是针对每一个线程的，每一个线程都拥有一个这样的全局变量 且 与其他线程隔离。
也就是说 新的线程调用此方法的时候 其 thread_arena 为 NULL 这个会在获取 arena 的函数中用到。
#### 2.3 malloc_init_state
对arena的初始化
```
#define NBINS             128
#define DEFAULT_MXFAST     (64 * SIZE_SZ / 4) // 16 * 8 = 128
// 
static void malloc_init_state (mstate av) {
    int i;
    mbinptr bin;

    /* Establish circular links for normal bins */
    // 初始化 arena的 bin 列表 bin的作用就是 一个缓冲区 方便查询
    for (i = 1; i < NBINS; ++i) {
        bin = bin_at (av, i);
        bin->fd = bin->bk = bin;
    }

    #if MORECORE_CONTIGUOUS
        if (av != &main_arena)
    #endif
    // 主线程arena 设置 XXX
            set_noncontiguous (av);
    // 主线程arena设置 fastbin的最大值
    if (av == &main_arena) set_max_fast (DEFAULT_MXFAST);
    atomic_store_relaxed (&av->have_fastchunks, false);
    av->top = initial_top (av);
}
```
函数 `malloc_init_state` 做的初始化功能主要有 初始化 arena 的 bins、如果是主线程 设置fastbin的最大值 以及 初始化 top
先看top的初始化
```
#define initial_top(M)              (unsorted_chunks (M))
#define unsorted_chunks(M)          (bin_at (M, 1))
```
top的初始化最终调用的还是 宏 `bin_at` 不同的是 top 固定了其index为1 所以我们重点关注 宏 `bin_at`

```
#define bin_at(m, i) (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2])) - offsetof (struct malloc_chunk, fd))
// 将其简化则如下
&bins[(i-1)*2] - 8*2

// bins 的定义
typedef struct malloc_chunk* mchunkptr;
mchunkptr bins[NBINS * 2 - 2];
```
从上面的定义关系中其实可以得出 `&bins[(i-1)*2]` 得出的其实是一个地址 在x64上占8个字节，后面的 `- 8*2` 相当于在数组上向左移动两个位置 最后我们得到 i 与 实际得到的值的关系是 `(i-2)*2` 结合代码
```
for (i = 1; i < NBINS; ++i) {
    bin = bin_at(av, i); // 这里指向 的是 bins 数组索引 (i-2)*2
    bin->fd = bin->bk = bin; // 这里再调用 bin->fd bin->bk 其实是在移动上一步得到的地址 通过 malloc_chunk 可以得出 fd 右位移2个位置 bk 右位移3个位置 这个时候就很容易理解为什么bin_at 的宏定义中要 ` - offsetof (struct malloc_chunk, fd)` 了
    // 也就是说通过bin_at 得到偏移后的地址 通过 bin->fb 正好又偏移回来了 也就是说 i = 1 时 idx 0 是bin_at(1)->fd idx 1 是bin_at(1)->bk
}
```
其实最终得到的 bins 的结构如下：
```
arr idx   0     1    2   3     4    5   ..............   249  250  251  252  253 254
        +---------+---------+---------+----------------+---------+---------+---------+
        | fd | bk | fd | bk | fd | bk | .............. | fd | bk | fd | bk | fd | bk |
        +---------+---------+---------+----------------+---------+---------+---------+
         bin_at(1) bin_at(2) bin_at(3)     bin_at(N)   bin_at(125) bin_at(126) bin_at(127) 
```
这样一个结构 将来是要承载 smallbin largebin unsortedbin 这三种bin链表的，初始化的时候并没有指向到一个malloc_chunk 地址 而是初始化了数组的地址 后面判断 一个bin是否为空链表的时候会用到这个进行判断


## 3. arena 的创建

arena创建这部分是整个malloc中比较重要的一部分了，在malloc代码通过宏 `arena_get` 实现，
```
#define arena_get(ptr, size) 
do {
    ptr = thread_arena;
    arena_lock (ptr, size);
} while (0)

#define arena_lock(ptr, size)
do {
    if (ptr) __libc_lock_lock (ptr->mutex);
    else ptr = arena_get2 ((size), NULL);
} while (0)

// 合并之后的代码
ptr = thread_arena;
// 如果当前线程arena不为空 则给当前线程加互斥锁
if (ptr) __libc_lock_lock (ptr->mutex);
// 当前线程arena为空 说明当前线程是刚刚创建 需要给当前线程创建arena
else ptr = arena_get2 ((size), NULL);
```

所以这里代码的重点是 给当前线程创建 arena 也就是调用方法 `arena_get2`

#### 3.1 arena_get2

```
static size_t narenas = 1;

static mstate arena_get2 (size_t size, mstate avoid_arena) {
    mstate a;
    static size_t narenas_limit;
    // 读取free_list
    a = get_free_list ();
    if (a == NULL) {
        /* Nothing immediately available, so generate a new arena.  */
        // 这里是给静态变量 narenas_limit 进行初始化值
        if (narenas_limit == 0) {
            // arena_max有值则使用arena_max的值
            if (mp_.arena_max != 0) narenas_limit = mp_.arena_max;
            else if (narenas > mp_.arena_test) {
                int n = __get_nprocs (); // 读取CPU核数
                if (n >= 1) narenas_limit = NARENAS_FROM_NCORES (n); // ((n) * (sizeof (long) == 4 ? 2 : 8)) -> cpu核数 * [2(x32)|8(x64)]
                else /* We have no information about the system.  Assume two cores.  */
                    narenas_limit = NARENAS_FROM_NCORES (2);
            }
        }
        repeat:;
        size_t n = narenas;
        if (__glibc_unlikely (n <= narenas_limit - 1)) {
            if (catomic_compare_and_exchange_bool_acq (&narenas, n + 1, n)) goto repeat;
            a = _int_new_arena (size);
            if (__glibc_unlikely (a == NULL)) catomic_decrement (&narenas);
        }
        else a = reused_arena (avoid_arena);
    }
    // 有空闲arena 则直接返回了
    return a;
}
```
这段代码主要有以下几个步骤
1. 通过函数 get_free_list 获取arena 如果获取到则直接返回 获取不到则继续往下进行
2. 通过内核数量计算出来arena数量最大值 narenas_limit = cpu核数 * 2/8 (x32/x64)
3. 通过 narenas 与 narenas_limit 确定是通过 `_int_new_arena` 创建新的 arena 还是 通过 reused_arena 利用现有的arena


接下来要着重分析的是 `get_free_list` `reused_arena` `_int_new_arena` 这三个函数 

#### 3.2 get_free_list 从free_list查找arena
从 free_list 中获取一个合适的 arena，free_list 是指向链表头部的指针，这个链表是由 malloc_state中的属性 `next_free` 链接的
```
// free_list 是一个malloc_state的指针 用以做成链表
typedef struct malloc_state *mstate;
static mstate free_list;

static mstate get_free_list (void) {
    // 当前线程arena
    mstate replaced_arena = thread_arena;
    mstate result = free_list;
    /*读取free_list 这个链表是从 fork调用 时候从父进程copy过来的，如果是在父进程中则free_list 一直为 NULL */
    if (result != NULL) {
        __libc_lock_lock (free_list_lock);
        // 将free_list 指向的malloc_state 从 malloc_state的next_free 组成的链表中取出
        result = free_list;
        if (result != NULL) {
            free_list = result->next_free;
            /* The arena will be attached to this thread.  */
            assert (result->attached_threads == 0);
            result->attached_threads = 1;
            detach_arena (replaced_arena); // 将当前线程指向的 malloc_state 中的属性 attached_threads 减一 因为下面要将当前线程换成从 free_list 中获取的arena
        }
        __libc_lock_unlock (free_list_lock);

        if (result != NULL) {
            LIBC_PROBE (memory_arena_reuse_free_list, 1, result);
            __libc_lock_lock (result->mutex);
            // 取出的arena作为当前线程的arena
            thread_arena = result;
        }
    }
    // 若free_list 为 NULL 则直接返回NULL
    return result;
}
// 如果 replaced_arena 不为空 则将其attached_threads 减一
static void detach_arena (mstate replaced_arena) {
    if (replaced_arena != NULL) {
        assert (replaced_arena->attached_threads > 0);
        /* The current implementation only detaches from main_arena in case of allocation failure.  
        This means that it is likely not beneficial to put the arena on free_list even if the reference count reaches zero.  */
        --replaced_arena->attached_threads;
    }
}
```
问题是 free_list 的数据是从哪里产生的呢，是从系统调用中的 `fork` 方法产生的，由于linux中线程之间是共享内存的 而进程之间是不共享的，使用 `fork`创建新的进程的时候 系统会复制一份父进程的内存到子进程中。系统的 `fork` 方法经过层层调用 最终调用的函数是 `__malloc_fork_unlock_child`

```
void __malloc_fork_unlock_child (void)
{
    // ptmalloc_init 没有调用过 也就是说 ptmalloc没有初始化
    if (__malloc_initialized < 1) return;
    /* Push all arenas to the free list, except thread_arena, which is attached to the current thread.  */
    __libc_lock_init (free_list_lock);
    if (thread_arena != NULL) thread_arena->attached_threads = 1;
    // 初始化 free_list 变量
    free_list = NULL;
    // 从主线程arena开始
    for (mstate ar_ptr = &main_arena;; ) {
        __libc_lock_init (ar_ptr->mutex);
        /*  free_list 作为一个链表的头指针，通过循环 arean_list 将他们组织到 一个 free_list 指引的链表上
            这些arena 都是从父进程中copy过来的 */
        if (ar_ptr != thread_arena) {
            /* This arena is no longer attached to any thread.  */
            ar_ptr->attached_threads = 0;
            // 通过 free_list 将这些arena通过next_free 串联成链表
            ar_ptr->next_free = free_list;
            free_list = ar_ptr;
        }
        // 移动到下一个 arena
        ar_ptr = ar_ptr->next;
        // 如果到了主线程arena 说明循环了一圈 结束循环
        if (ar_ptr == &main_arena) break;
    }

    __libc_lock_init (list_lock);
}
```

#### 3.3 reused_arena 重用arena
这个方法是从当前进程中的 arena 列表中查询可用的arena 进行接下来的内存分配。其思路是 先通过main_arena 为起点循环遍历 arena 链表，对链表中的arena进行加锁尝试 如果失败则循环到下一个arena 如果成功 将当前线程的arena引用变量 thread_arena替换为当前的arena。
替换过程需要做一定的处理 
1. 替换前的thread_arena 的属性 attached_thread需要减一 
2. 将当前arena从free_list 中移除 将当前arena的attached_thread 加一 
3. 给 thread_arena赋值并返回 最后还会移动下 next_to_use 方便下次使用
```

static mstate reused_arena (mstate avoid_arena) {
    mstate result;
    /* FIXME: Access to next_to_use suffers from data races.  */
    // 初始化静态变量 next_to_use 为 &main_arena
    static mstate next_to_use;
    if (next_to_use == NULL) next_to_use = &main_arena;

    /* Iterate over all arenas (including those linked from free_list).  */
    result = next_to_use;
    // 从main_arena开始循环arena链表 直到循环一圈停止查询。在循环过程中 尝试给arena加锁 加锁成功则执行 out 代码片段，另外 __libc_lock_trylock 不会等待锁的 拿不到锁会立刻返回
    do {
        if (!__libc_lock_trylock (result->mutex)) goto out;
        /* FIXME: This is a data race, see _int_new_arena.  */
        result = result->next;
    }while (result != next_to_use);

    /* Avoid AVOID_ARENA as we have already failed to allocate memory in that arena and it is currently locked.   */
    if (result == avoid_arena) result = result->next;

    /* No arena available without contention.  Wait for the next in line.  */
    LIBC_PROBE (memory_arena_reuse_wait, 3, &result->mutex, result, avoid_arena);
    __libc_lock_lock (result->mutex);

    /* Attach the arena to the current thread.  */
    out:
        {
            /* Update the arena thread attachment counters.   */
            mstate replaced_arena = thread_arena;
            __libc_lock_lock (free_list_lock);
            detach_arena (replaced_arena); // 将这个arena的attached_thread 减一 因为thread_arena 需要更换
            // 将result从free_list 中去除
            remove_from_free_list (result);
            // 新的thread_arean 加一
            ++result->attached_threads;
            __libc_lock_unlock (free_list_lock);
        }

    LIBC_PROBE (memory_arena_reuse, 2, result, avoid_arena);
    // 替换 thread_arena
    thread_arena = result;
    // 将 next_to_use 移动到下一个
    next_to_use = result->next;

    return result;
}

static void remove_from_free_list (mstate arena) {
    mstate *previous = &free_list;
    // 循环 free_list
    for (mstate p = free_list; p != NULL; p = p->next_free) {
        assert (p->attached_threads == 0);
        if (p == arena) { // 在free_list 的链表上找到参数 arena
            /* Remove the requested arena from the list.  */
            *previous = p->next_free;
            break;
        }
        else previous = &p->next_free;
    }
}
```

#### 3.4 `_int_new_arena` 新建arena
新建一个arena 每个arena都是有若干个 heap 组成的 heap 通过 `new_heap` 创建，然后就是要设置一些 malloc_state 的参数
重要的是 这里创建的 top 后面的内存分配会从top上切割产生，后面有了 bins 的缓存则会从缓存中获取
```
static mstate _int_new_arena (size_t size) {
    mstate a;
    heap_info *h;
    char *ptr;
    unsigned long misalign;
    // 创建 sub_heap 其实 new_heap 是申请内存 这里将 malloc_state top_pad heap_info 和 size 全包含进去 申请内存了
    // 多申请 MALLOC_ALIGNMENT 这段内存的目的是 下面对齐的时候需要用到的 怕内存不够
    h = new_heap (size + (sizeof (*h) + sizeof (*a) + MALLOC_ALIGNMENT), mp_.top_pad);
    if (!h) {
        // 创建失败 再次尝试创建 再不成功直接返回
        h = new_heap (sizeof (*h) + sizeof (*a) + MALLOC_ALIGNMENT, mp_.top_pad);
        if (!h) return 0;
    }
    // 给heap设置 arena 指针
    a = h->ar_ptr = (mstate) (h + 1);
    // 初始化 arena的bins
    malloc_init_state (a);
    a->attached_threads = 1;
    /*a->next = NULL;*/
    a->system_mem = a->max_system_mem = h->size;

    /* Set up the top chunk, with proper alignment. */
    // 新建的arena 初始化arena 初始化top且内存尺寸对齐
    // 这里可以看到 当新建一个arena的时候其里面的chunk只有一个top chunk
    ptr = (char *) (a + 1);
    // misalign 是向下对齐多出来的长度
    misalign = (unsigned long) chunk2mem (ptr) & MALLOC_ALIGN_MASK;
    // 先上对齐
    if (misalign > 0) ptr += MALLOC_ALIGNMENT - misalign;
    // 设置 top chunk
    top (a) = (mchunkptr) ptr;
    set_head (top (a), (((char *) h + h->size) - ptr) | PREV_INUSE);

    LIBC_PROBE (memory_arena_new, 2, a, size);
    // 切换 thread_arena 
    mstate replaced_arena = thread_arena;
    thread_arena = a;
    
    // 当前新的arena 加锁 添加到arena链表 
    __libc_lock_init (a->mutex);
    __libc_lock_lock (list_lock);
    // 添加arena到链表
    a->next = main_arena.next;
    atomic_write_barrier ();
    main_arena.next = a;
    __libc_lock_unlock (list_lock);
    __libc_lock_lock (free_list_lock);
    detach_arena (replaced_arena);
    __libc_lock_unlock (free_list_lock);
    // arena增加互斥锁
    __libc_lock_lock (a->mutex);

    return a;
}
```

着重看下 在 `new_heap` 申请完内存后 在函数 `_int_new_arena` 中对这段内存的划分部分 以及top 部分
```
// h 指向 new_heap 申请到的内存地址起始位置 并且类型是被转成 heap_info 后返回的，这里 h+1 正好是到 h分配完类型 heap_info 成员占用内存后的位置
// 且通过类型转换 将后面的内存又划分出了一部分malloc_state 的内存
a = h->ar_ptr = (mstate) (h + 1);
ptr = (char *) (a + 1);
// 这里是做内存对齐的操作
misalign = (unsigned long) chunk2mem (ptr) & MALLOC_ALIGN_MASK;
if (misalign > 0) ptr += MALLOC_ALIGNMENT - misalign;
// 开始分配top部分
top (a) = (mchunkptr) ptr;
set_head (top (a), (((char *) h + h->size) - ptr) | PREV_INUSE);
```
内存划分后的情况如下图所示：
![内存分配情况](WX20190517-150525@2x.png)
对于top部分的处理 宏 进行展开分析
```
#define top(ar_ptr) ((ar_ptr)->top)
#define set_head(p, s)       ((p)->mchunk_size = (s))

// 对应代码展开如下
ptr = (char *) (a + 1);
a->top = (mchunkptr) ptr; // 确定指针位置
// h + size - ptr 这就是剩下的空闲空间的长度
a->top->mchunk_size = (((char *) h + h->size) - ptr) | PREV_INUSE; // 设置 top chunk 的 mchunk_size 字段 设置chunk大小 以及 设置 prev_inuse 为已使用(其实这个时候 top chunk 前面并没有 chunk ，设置为已使用是为了后面chunk合并的时候 有一个结束条件)
```

#### 3.5 `new_heap` 新建arena 的 sub_heap
每个arena 是 由多个行程链表的 heap组成的 称作 sub_heap 吧，其创建的整体逻辑如下注释: 
```
static heap_info * new_heap (size_t size, size_t top_pad) {
    // 获取系统内存管理的基本尺寸 一般为 4K 这个值可以在 glibc中的全局变量中找到
    size_t pagesize = GLRO (dl_pagesize); // 4K
    char *p1, *p2;
    unsigned long ul;
    heap_info *h;
    /*
    对size进行处理
    如果size + top_pad 小于 HEAP_MIN_SIZE 则使用最小值
    如果size + top_pad 小于或等于HEAP_MAX_SIZE 则size = size + top_pad
    如果size 大于 HEAP_MAX_SIZE 则 return 0 
    */
    if (size + top_pad < HEAP_MIN_SIZE) size = HEAP_MIN_SIZE;
    else if (size + top_pad <= HEAP_MAX_SIZE) size += top_pad;
    else if (size > HEAP_MAX_SIZE) return 0;
    else size = HEAP_MAX_SIZE;
    size = ALIGN_UP (size, pagesize); // 4K对齐

    p2 = MAP_FAILED;
    // 如果上次分配内存计算出了下次内存分配的起始地址 这里合一直接分配了 减少一定的计算量与内核态的切换
    if (aligned_heap_area) {
        // 直接基于aligned_heap_area作为起始地址分配
        p2 = (char *) MMAP (aligned_heap_area, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
        // 清除了
        aligned_heap_area = NULL;
        // 分配失败 返还申请的内存
        if (p2 != MAP_FAILED && ((unsigned long) p2 & (HEAP_MAX_SIZE - 1))) {
            __munmap (p2, HEAP_MAX_SIZE);
            p2 = MAP_FAILED;
        }
    }
    /*
    这里通过mmap申请内存，成功会返回申请的内存段的起始地址。且这里想要将内存地址的起始与结束的地址是与 HEAP_MAX_SIZE 对齐的 内存长度是 HEAP_MAX_SIZE。
    这里会看到先申请了两倍 HEAP_MAX_SIZE 尺寸的内存，这是怕 当返回的地址不是内存对齐的，这个时候需要向后调整内存地址 使内存起始地址与 HEAP_MAX_SIZE 对齐
    这里向后是因为前面的地址可能已经被其他程序申请，然后又要保持内存长度是 HEAP_MAX_SIZE ，这样前面申请的 HEAP_MAX_SIZE 长度内存就不够了，需要再次调用mmap
    为了减少调用 这里申请了两倍的内存 然后将多余的通过 __munmap 还回去
    */
    if (p2 == MAP_FAILED) {
        // 申请 两倍 HEAP_MAX_SIZE 长度的内存 p1是内核返回的内存起始地址
        p1 = (char *) MMAP (0, HEAP_MAX_SIZE << 1, PROT_NONE, MAP_NORESERVE);
        if (p1 != MAP_FAILED) { // 分配成功 对内存尺寸进行调整 使其在 HEAP_MAX_SIZE长度的内存段上
            // p1 针对HEAP_MAX_SIZE向上对齐
            p2 = (char *) (((unsigned long) p1 + (HEAP_MAX_SIZE - 1)) & ~(HEAP_MAX_SIZE - 1));
            ul = p2 - p1;//由于是向上 所以是 p2 - p1，若 ul = 0 说明 mmap分配的起始地址恰好是HEAP_MAX_SIZE对齐的，不然就需要调整 
            if (ul) __munmap (p1, ul); // 返还起始地址向上偏移后多出来的地址
            else aligned_heap_area = p2 + HEAP_MAX_SIZE;
            // 返还由于 两倍HEAP_MAX_SIZE 多出来的部分 只需要 HEAP_MAX_SIZE 长度的内存就好了
            __munmap (p2 + HEAP_MAX_SIZE, HEAP_MAX_SIZE - ul);
        }
        else {
            /* Try to take the chance that an allocation of only HEAP_MAX_SIZE is already aligned. */
            // 两倍内存申请不到，这里只好申请 HEAP_MAX_SIZE 内存 碰碰运气了
            p2 = (char *) MMAP (0, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
            if (p2 == MAP_FAILED) return 0; // 分配失败
            // 运气不好 返回的地址 并不能与HEAP_MAX_SIZE对齐 这里只好返回申请的内存 并结束
            if ((unsigned long) p2 & (HEAP_MAX_SIZE - 1)) {
                __munmap (p2, HEAP_MAX_SIZE);
                return 0;
            }
        }
    }
    // 设置读写权限
    if (__mprotect (p2, size, PROT_READ | PROT_WRITE) != 0) {
        __munmap (p2, HEAP_MAX_SIZE);
        return 0;
    }
    // 设置一些heap参数 返回这个申请的heap
    h = (heap_info *) p2;
    h->size = size;
    h->mprotect_size = size;
    LIBC_PROBE (memory_heap_new, 2, h, h->size);
    return h;
}
```
先看参数处理部分，相关的宏定义如下：
```
#define HEAP_MIN_SIZE (32 * 1024) // 32K
#define HEAP_MAX_SIZE (2 * DEFAULT_MMAP_THRESHOLD_MAX) // x64 上是64M
#define DEFAULT_MMAP_THRESHOLD_MAX (4 * 1024 * 1024 * sizeof(long)) // 4 * [4|8] M
```
保持size的大小在 HEAP_MIN_SIZE 与 HEAP_MAX_SIZE 之间

然后是 `size_t pagesize = GLRO (dl_pagesize);` 展开宏
```
#define GLRO(name) _##name // _dl_pagesize
/* Cached value of `getpagesize ()'.  */
EXTERN size_t _dl_pagesize;
size_t _dl_pagesize = EXEC_PAGESIZE;
#define EXEC_PAGESIZE   4096

// 最终的形式是
size_t pagesize = EXEC_PAGESIZE;// 4096 byte  也就是 4k
```

然后是补齐的算法 
```
// 补齐算法 size + size&(base-1)
#define ALIGN_UP(base, size)    ALIGN_DOWN ((base) + (size) - 1, (size))
#define ALIGN_DOWN(base, size)  ((base) & -((__typeof__ (base)) (size))) // base & -size
```

这段主要是通过 mmap 申请内存 使用到的一个重要的方法是 mmap 函数原型 `void* mmap(void* start, size_t length, int prot, int flags, int fd, off_t offset)` 和 `int munmap(void* start, size_t length)` ，mmap 分配内存 munmap 释放内存，这些操作都是要切到内核态执行的，
|参数|例子|意义|
|--|--|--|
|void* start|NULL|内存分配开始的位置 写NULL 这个位置由内核决定|
|size_t length|[number]|分配内存的长度|
|int prot|PROT_EXEC:执行 PROT_READ:读 PROT_WRITE:写 PROT_NONE:不可访问|权限|
|int flags||文件类型 需要与 open 的时候使用的配对儿|
|int fd|-1|文件句柄 -1 是与匿名搭配使用|
|off_t offset|0|写入文件的偏移量|
这里代码逻辑的设计也是为了减少内核函数的调用，先看其如何通过一系列宏调用系统函数的
```
#define MMAP_CALL(__nr, __addr, __len, __prot, __flags, __fd, __offset) INLINE_SYSCALL_CALL (__nr, __addr, __len, __prot, __flags, __fd, __offset)
#define INLINE_SYSCALL_CALL(...) __INLINE_SYSCALL_DISP (__INLINE_SYSCALL, __VA_ARGS__)
#define __INLINE_SYSCALL_DISP(b,...) __SYSCALL_CONCAT (b,__INLINE_SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

#define __INLINE_SYSCALL_NARGS(...) __INLINE_SYSCALL_NARGS_X (__VA_ARGS__,7,6,5,4,3,2,1,0,)
#define __INLINE_SYSCALL_NARGS_X(a,b,c,d,e,f,g,h,n,...) n

// 以上宏展开情况
__SYSCALL_CONCAT (
    __INLINE_SYSCALL,
    1
    )(__VA_ARGS__)

#define __SYSCALL_CONCAT(a,b)       __SYSCALL_CONCAT_X (a, b)
#define __SYSCALL_CONCAT_X(a,b)     a##b

// 得到
__INLINE_SYSCALL1(__VA_ARGS__)

#define __INLINE_SYSCALL1(name, a1) INLINE_SYSCALL (name, 1, a1)
#define INLINE_SYSCALL(name, nr, args...) __syscall_##name (args)

// 最终调用的内核方法是
__syscall_mmap(__addr, __len, __prot, __flags, __fd, __offset)

```
其中有一些宏在做 `##` 进行字符拼接的时候 做了两层嵌套，其原因是 在 `##` 连接的宏定义中 里面的字符不会在进行宏展开了，做一层嵌套的目的是 在前一层先做宏展开 然后在发给下一个宏做 `##` 以此实现宏展开的需要。
最终调用的方法是 `__syscall_mmap`

然后是mmap分配内存的逻辑，先抽离出主要逻辑功能 主要分三个部分
```
static char *aligned_heap_area;
p2 = MAP_FAILED;
if (aligned_heap_area) {
    p2 = (char *) MMAP (aligned_heap_area, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
    //....#1...
}
if (p2 == MAP_FAILED) {
    p1 = (char *) MMAP (0, HEAP_MAX_SIZE << 1, PROT_NONE, MAP_NORESERVE);
    if (p1 != MAP_FAILED) { 
        //...#2...
    }
    else {
        p2 = (char *) MMAP (0, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
        // ...#3....
    }
}
```
如果存在 aligned_heap_area 则按照 aligned_heap_area 作为内存开始地址进行扩展（#1 处代码），否则按照两倍的 HEAP_MAX_SIZE 进行内存的申请（#2 处代码），申请失败的话 再次尝试申请 HEAP_MAX_SIZE 的内存（#3 处代码）。由于mmap 返回的是申请成功的内存的起始位置，这里分配内存 有一个需求是：如果将内存地址(虚拟地址) 按照 HEAP_MAX_SIZE 的长度进行分段 ，new_heap 分配的内存正好在这个段上 起始地址和结束地址都是 HEAP_MAX_SIZE 对齐的 且 长度是 HEAP_MAX_SIZE。
当内存申请成功后，这个时候是有申请内存的结尾地址的，将这个地址赋值给 aligned_heap_area 方便下次分配内存的时候 可以直接确定分配内存开始位置 见代码 #1 处，但是在这个过程中内存很有可能被其他线程或进程申请到，这个变量就没有用了 也就是 #1 处的申请内存的代码会失败，
这个场景可能比较有限 所以代码 #1 处使用完后立即就将这个变量赋值为NULL了，这个场景可能比较适用连续的大段内存分配吧。

重点是 #2 处的代码，他会尝试分配 两倍 HEAP_MAX_SIZE 的内存，可能会奇怪为什么分配两段。ptmalloc 在通过new_heap 申请内存的时候 他想要申请一段长度为 HEAD_MAX_SIZE 长度的内存，且这段内存的起始内存地址与结束地址是与 HEAD_MAX_SIZE 对齐的，也就是说 分配到的内存地址起始是 `n * HEAD_MAX_SIZE` 结束地址是 `(n+1) * HEAD_MAX_SIZE`。
对于 mmap 函数 第一个参数是起始地址，一般会给 NULL 让内核决定起始地址，这样内核返回的地址很大情况不会是 HEAD_MAX_SIZE 的倍数，这个时候需要计算返回的内存地址的对齐地址，当然是需要向上对齐，因为前面的内存已经被其他线程占用了。如果我们申请的长度是 HEAD_MAX_SIZE ，如果返回的起始地址不对齐 就要再次申请一段内存补充整个内存长度为 HEAD_MAX_SIZE 但是当再次申请的时候有可能会被其他线程前先一步 这个时候补充的内存段与之前申请的内存段并不连续了，这样就很糟糕。所以ptmallloc在这里一次申请 `2 * HEAD_MAX_SIZE` 的内存 然后通过`__munmap`函数将多余的内存返还给系统，通过这个思路来看代码就很好理解了，可以参考下图：
![内存的切割内存](WX20190518-230344@2x.png)
其中 p1 是mmap返回的地址 ，p1 以 HEAD_MAX_SIZE 向上对齐得到 p2 ，p2是我们想要的内存起始地址，同样需要将 p1 开始 长度 p2 - p1 的内存返还给系统，p2 - p1 也就是 ul。
同时还有尾部多余的内存需要返还给系统，我们最终需要的内存的结束内存地址是 `p2 + HEAP_MAX_SIZE` 这也是需要返还给系统的内存起始地址，然后是长度。申请两倍内存的结束地址是 `p1 + 2 * HEAP_MAX_SIZE` 得到尾部需要返还的内存长度是 `(p1 + 2 * HEAP_MAX_SIZE) - (p2 + HEAP_MAX_SIZE)` ，结合 `p2 - p1 = ul` 最终得到 `HEAP_MAX_SIZE - ul` 这也就是代码中的尾部返还系统内存的长度。

```
p1 = (char *) MMAP (0, HEAP_MAX_SIZE << 1, PROT_NONE, MAP_NORESERVE);
if (p1 != MAP_FAILED) { // 成功分配两倍HEAP_MAX_SIZE长度的内存 p1 是内存起始地址
    // p2 是 p1 以HEAP_MAX_SIZE向上对齐的地址
    p2 = (char *) (((unsigned long) p1 + (HEAP_MAX_SIZE - 1)) & ~(HEAP_MAX_SIZE - 1));
    ul = p2 - p1; // p1 需要向上补齐的长度
    // 将头部多余的内存返还给系统
    if (ul) __munmap (p1, ul);
    // 返回的内存地址正好是对齐的 这时给变量aligned_heap_area赋值 给下一个连续的内存申请用 可以直接确定内存起始位置
    else aligned_heap_area = p2 + HEAP_MAX_SIZE;
    // 将尾部多余的内存返还给系统
    __munmap (p2 + HEAP_MAX_SIZE, HEAP_MAX_SIZE - ul);
}
```

然后是 #3 处的代码，这里的代码好理解，当上一步分配两倍HEAP_MAX_SIZE长度内存的时候失败的话，这里尝试申请HEAP_MAX_SIZE长度内存，看看返回的地址是否为 以HEAP_MAX_SIZE对齐的起始内存地址，这里纯粹是在碰运气的。
```
// 申请 HEAP_MAX_SIZE 长度内存
p2 = (char *) MMAP (0, HEAP_MAX_SIZE, PROT_NONE, MAP_NORESERVE);
if (p2 == MAP_FAILED) return 0; // 分配失败 返回
// 分配的内存起始地址 p2 并不是以HEAP_MAX_SIZE对齐的 则返还内存 直接返回
if ((unsigned long) p2 & (HEAP_MAX_SIZE - 1)) {
    __munmap (p2, HEAP_MAX_SIZE);
    return 0;
}
```



## 4. 内存的申请过程总结
