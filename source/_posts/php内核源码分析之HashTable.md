---
title: php内核源码分析之HashTable
tags: [php,php-src,内核源码]
categories: [php]
date: 2019-05-26 21:35:59
---
HashTable是一个常用的数据结构，在php内核对HashTable的使用很多，我们熟知的 array 在内核中的实现形式其实也是 HashTable，包括在其他语言中存在的数据结构 `LinkList` ，在php中已经将这些基本数据类型屏蔽 对用户只以 `array` 的形式暴漏，理解php内核中的hashtable对理解php数组十分重要。
<!-- more -->

## 1. 基本数据结构
在php中数据类型是通过 `struct` 体现的，对于HashTable 也是以array类型体现的，看下其代码结构:
```
typedef struct _zend_array HashTable;
typedef struct _zend_array      zend_array;
```
其中 `zend_array` `HashTable` 对应的都是结构体 `struct _zend_array`，结构题代码如下:
```
struct _zend_array {
    zend_refcounted_h gc; // 引用次数记录 gc回收使用
    union {
        struct {
            ZEND_ENDIAN_LOHI_4(
                zend_uchar    flags,
                zend_uchar    _unused,
                zend_uchar    nIteratorsCount,
                zend_uchar    _unused2)
        } v;
        uint32_t flags;
    } u; // 这个联合体记录 数据类型相关的信息 
    uint32_t          nTableMask;           // 掩码 计算hashcode 映射区索引
    Bucket           *arData;               // 指针 指向 Bucket 类型数组的 内存地址
    uint32_t          nNumUsed;             // 每新增一次这个属性会计数加一 删除的时候这个值不会减
    uint32_t          nNumOfElements;       // 哈希表中实际存储的可用元素 与 nNumUsed的区别是 这个属性会在删除元素的时候减一
    uint32_t          nTableSize;           // 哈希表中能存储的元素总数 这个值总是与 8 对齐的
    uint32_t          nInternalPointer;     // 内部迭代计数器相关的函数有 next prev end reset current
    zend_long         nNextFreeElement;     // ？？？
    dtor_func_t       pDestructor;          // 析构函数地址 算是个钩子函数 当哈希表被销毁的时候被调用
};
```
这就是一个哈希表的数据类型，但是其核心的数据存储在 指针 `arData` 那里，需要看下 `Bucket` 这个类型 其实Bucket存储的是一个键值对
```
typedef struct _Bucket {
    zval              val;  // 值部分 类型是zval 也就是 struct _zval_struct
    zend_ulong        h;    // 键 字符串的 hashcode 值
    zend_string      *key;  // 哈希表中一个键值对的 键 类型字符串
} Bucket;
```
以上是 存储一个键值对的 Bucket，主要是值部分需要看下，zval 类型就是 `struct _zval_struct` 则是 php中数据类型的总类型，其中在哈希表中常用的有以下两个属性
```
zval.value      // 这个是 Bucket 中 值部分的真实值的存储地方
zval.u2.next    // 这个是 哈希表中用以构成冲突链表 存储下一个元素存储位置的索引
```
具体行程的链表形式看下面

## 2. 初始化创建
在php中创建数组可以使用宏 `zend_new_array` 最终的调用链如下
```
#define zend_new_array(size) _zend_new_array(size)

ZEND_API HashTable* ZEND_FASTCALL _zend_new_array(uint32_t nSize)
{
    // 申请内存 存储 HashTable
    HashTable *ht = emalloc(sizeof(HashTable));
    // 初始化
    _zend_hash_init_int(ht, nSize, ZVAL_PTR_DTOR, 0);
    return ht;
}

// 初始化一个 HashTable
static zend_always_inline void _zend_hash_init_int(HashTable *ht, uint32_t nSize, dtor_func_t pDestructor, zend_bool persistent)
{
    GC_SET_REFCOUNT(ht, 1);
    GC_TYPE_INFO(ht) = IS_ARRAY | (persistent ? (GC_PERSISTENT << GC_FLAGS_SHIFT) : (GC_COLLECTABLE << GC_FLAGS_SHIFT));
    // 设置为未初始化
    HT_FLAGS(ht) = HASH_FLAG_UNINITIALIZED;
    // 掩码
    ht->nTableMask = HT_MIN_MASK;
    // 设置 arData 的地址
    HT_SET_DATA_ADDR(ht, &uninitialized_bucket);
    // 以下属性 上面讲过其代表意义 这里统一初始化为 0
    ht->nNumUsed = 0;
    ht->nNumOfElements = 0;
    ht->nInternalPointer = 0;
    ht->nNextFreeElement = 0;
    ht->pDestructor = pDestructor;
    // 计算哈希表的尺寸
    ht->nTableSize = zend_hash_check_size(nSize);
}
```
着重看下初始化的几个重要步骤
1. 设置掩码，掩码设置的值是一个宏 `#define HT_MIN_MASK ((uint32_t) -2)` 是无符号的 -2 
2. 设置数据部分的地址 调用的也是宏，涉及到的宏如下
```
HT_SET_DATA_ADDR(ht, &uninitialized_bucket);

// 定义的初始化的一个数组 长度为2 默认值是 无符号的-1 也就是所有位都是1 
#define HT_INVALID_IDX ((uint32_t) -1)
static const uint32_t uninitialized_bucket[-HT_MIN_MASK] =
    {HT_INVALID_IDX, HT_INVALID_IDX};
// 这里是对哈希表中的 arData进行赋值
#define HT_SET_DATA_ADDR(ht, ptr) do { \
        (ht)->arData = (Bucket*)(((char*)(ptr)) + HT_HASH_SIZE((ht)->nTableMask)); \
    } while (0)
#define HT_HASH_SIZE(nTableMask) \
    (((size_t)(uint32_t)-(int32_t)(nTableMask)) * sizeof(uint32_t))
```
重点需要看下给 arData 赋值地址的过程，宏进行展开后的结果是`ht->arData = (Bucket*)(ptr + (uint32_t)(-1 * ht->nTableMask) * sizeof(unit32_t));` ，如果按照初始化时候 mask 为 -2 进行计算的话 arData正好是 数组 uninitialized_bucket 的结尾处

这里哈希映射区是 数据区的两倍 目的是为了减少 哈希冲突，相当于 php中哈希的平衡因子是 0.5。

3. 计算哈希表的尺寸 调用了函数 `zend_hash_check_size`

```
// 哈希表最小尺寸
#define HT_MIN_SIZE 8
// 哈希表最大尺寸
#define HT_MAX_SIZE 0x80000000

static zend_always_inline uint32_t zend_hash_check_size(uint32_t nSize)
{
    /* Use big enough power of 2 */
    /* size should be between HT_MIN_SIZE and HT_MAX_SIZE */
    // 如果用户申请的尺寸小于等于 宏定义的最小尺寸 则使用宏定义的最小尺寸
    if (nSize <= HT_MIN_SIZE) {
        return HT_MIN_SIZE;
    } else if (UNEXPECTED(nSize >= HT_MAX_SIZE)) {
        // 超出定义的最大尺寸则报错了
        zend_error_noreturn(E_ERROR, "Possible integer overflow in memory allocation (%u * %zu + %zu)", nSize, sizeof(Bucket), sizeof(Bucket));
    }
    // 尺寸以8对齐
    return 0x2 << (__builtin_clz(nSize - 1) ^ 0x1f);
}
```
#### 2.1 哈希表数据部分的存储结构
对于上面的结构刚开始是很奇怪的，注意这里代码是 php7.4 的，这个版本以下的实现方式并不一样。这里可以看下php的哈希表的设计思路了，用户存储哈希表数据的内存地址被分为两个部分，一部分是数据部分存储元素地址 `Bucket *` 另一部分是 hash值与数据部分索引的关系，其结构如下图所示：
![哈希表数据部分结构](20190530195834.png)
这段内存可以分为两个部分 `数据部分` 和 `hash映射部分`，整个哈希表的操作都是基于这个结构进行计算的。看看这段内存有什么特征：
1. 整个内存在真正开始使用的时候才分配 分配的时候是将两部分所需内存加在一起申请的
2. ht->arData 所指向的内存地址正好是存储数据部分的内存地址的开始位置
3. hash映射部分存储的数据个数是数据部分的两倍 但是两者占用的内存空间是一样的，哈希部分存储的数据类型是 uint32_t 4个字节 数据部分存储的是指针 也就是内存地址 内存地址是 8字节

了解了这个结构 再看看 如何使用的，以添加元素为例：
1. 得到键值对 计算键字符串的哈希值
2. 将键值对 Bucket 顺序存储到 数据部分
3. 通过哈希值与掩码 ht->nTableMask 与 运算 计算出哈希映射区的索引
4. 形成链表 映射部分的值赋值给数据部分的 next指针 (zval.u2.next) , 将数据的索引值 赋值给映射区对应内存块

以上第 4 步 在形成冲突的时候，会被映射到相同的 映射区域 ，好在 通过 zval.u2.next 可以形成冲突链表，具体实现看下一部分
## 3. 添加或更新元素
添加元素与更新元素的区别是 更新元素需要先查找，所以这部分可以分为 添加 与 查找两部分，实现这个过程的代码是下面这个函数
```
static zend_always_inline zval *_zend_hash_add_or_update_i(HashTable *ht, zend_string *key, zval *pData, uint32_t flag)
{
    zend_ulong h;
    uint32_t nIndex;
    uint32_t idx;
    Bucket *p, *arData;

    IS_CONSISTENT(ht);
    HT_ASSERT_RC1(ht);
    // #1 初始化
    if (UNEXPECTED(HT_FLAGS(ht) & (HASH_FLAG_UNINITIALIZED|HASH_FLAG_PACKED))) {
        if (EXPECTED(HT_FLAGS(ht) & HASH_FLAG_UNINITIALIZED)) {
            zend_hash_real_init_mixed(ht);
            if (!ZSTR_IS_INTERNED(key)) {
                zend_string_addref(key);
                HT_FLAGS(ht) &= ~HASH_FLAG_STATIC_KEYS;
                zend_string_hash_val(key);
            }
            goto add_to_hash;
        } else {
            zend_hash_packed_to_hash(ht);
            if (!ZSTR_IS_INTERNED(key)) {
                zend_string_addref(key);
                HT_FLAGS(ht) &= ~HASH_FLAG_STATIC_KEYS;
                zend_string_hash_val(key);
            }
        }
    } else if ((flag & HASH_ADD_NEW) == 0) {
        #2 更新数据 先查找
        p = zend_hash_find_bucket(ht, key, 0);

        if (p) {
            zval *data;

            if (flag & HASH_ADD) {
                if (!(flag & HASH_UPDATE_INDIRECT)) {
                    return NULL;
                }
                ZEND_ASSERT(&p->val != pData);
                data = &p->val;
                if (Z_TYPE_P(data) == IS_INDIRECT) {
                    data = Z_INDIRECT_P(data);
                    if (Z_TYPE_P(data) != IS_UNDEF) {
                        return NULL;
                    }
                } else {
                    return NULL;
                }
            } else {
                ZEND_ASSERT(&p->val != pData);
                data = &p->val;
                if ((flag & HASH_UPDATE_INDIRECT) && Z_TYPE_P(data) == IS_INDIRECT) {
                    data = Z_INDIRECT_P(data);
                }
            }
            if (ht->pDestructor) {
                ht->pDestructor(data);
            }
            ZVAL_COPY_VALUE(data, pData);
            return data;
        }
        if (!ZSTR_IS_INTERNED(key)) {
            zend_string_addref(key);
            HT_FLAGS(ht) &= ~HASH_FLAG_STATIC_KEYS;
        }
    } else if (!ZSTR_IS_INTERNED(key)) {
        zend_string_addref(key);
        HT_FLAGS(ht) &= ~HASH_FLAG_STATIC_KEYS;
        zend_string_hash_val(key);
    }

    #3 数组扩容
    ZEND_HASH_IF_FULL_DO_RESIZE(ht);        /* If the Hash table is full, resize it */
#4 添加新元素
add_to_hash:
    idx = ht->nNumUsed++;
    ht->nNumOfElements++;
    arData = ht->arData;
    p = arData + idx;
    p->key = key;
    p->h = h = ZSTR_H(key);
    nIndex = h | ht->nTableMask;
    Z_NEXT(p->val) = HT_HASH_EX(arData, nIndex);
    HT_HASH_EX(arData, nIndex) = HT_IDX_TO_HASH(idx);
    ZVAL_COPY_VALUE(&p->val, pData);

    return &p->val;
}
```
其核心部分在代码里已经标明 分别是 `#1 初始化 #2 更新数据 先查找 #3 数组扩容 #4 添加新元素`，其中扩容部分在下一部分单独讲，下面分步分析
#### 3.1 初始化
初始化部分代码设计到的两个函数 `zend_hash_real_init_mixed` `zend_hash_packed_to_hash` 对于 `zend_hash_real_init_mixed`最终调用的函数是 `zend_hash_real_init_mixed_ex`
```
static zend_always_inline void zend_hash_real_init_mixed_ex(HashTable *ht)
{
    void *data;
    uint32_t nSize = ht->nTableSize;

    if (UNEXPECTED(GC_FLAGS(ht) & IS_ARRAY_PERSISTENT)) {
        data = pemalloc(HT_SIZE_EX(nSize, HT_SIZE_TO_MASK(nSize)), 1);
    } else if (EXPECTED(nSize == HT_MIN_SIZE)) {
        data = emalloc(HT_SIZE_EX(HT_MIN_SIZE, HT_SIZE_TO_MASK(HT_MIN_SIZE)));
        // ......
        return;
    } else {
        // 申请一段新的内存
        data = emalloc(HT_SIZE_EX(nSize, HT_SIZE_TO_MASK(nSize)));
    }
    // 计算掩码
    ht->nTableMask = HT_SIZE_TO_MASK(nSize);
    // 给arData重新赋值 这时其实已经抛弃了哈希表初始化时候使用的 uninitialized_bucket
    HT_SET_DATA_ADDR(ht, data);
    // 设置类型
    HT_FLAGS(ht) = HASH_FLAG_STATIC_KEYS;
    // 初始化
    HT_HASH_RESET(ht);
}
```
逻辑过程不复杂 就是申请内存 初始化哈希表的一些值，为了更好的理解代码 还是要对这些宏展开分析
```
// 计算整个哈希表需要的内存空间 单位 byte，其实就是数据部分与哈希映射部分的内存空间只和
#define HT_SIZE_EX(nTableSize, nTableMask) \
    (HT_DATA_SIZE((nTableSize)) + HT_HASH_SIZE((nTableMask)))
// 哈希映射部分需要的内存空间
#define HT_HASH_SIZE(nTableMask) \
    (((size_t)(uint32_t)-(int32_t)(nTableMask)) * sizeof(uint32_t))
// 数据部分需要的内存空间
#define HT_DATA_SIZE(nTableSize) \
    ((size_t)(nTableSize) * sizeof(Bucket))
// 通过哈希表尺寸计算对应的掩码 其实就是 (unsigned int)(-2 * nTableSize)
#define HT_SIZE_TO_MASK(nTableSize) \
    ((uint32_t)(-((nTableSize) + (nTableSize))))
// 设置 arData的值 通过上面的分析 arData的值是数据部分开始的内存地址
// 这里就是通过 申请到的内存开始地址 ptr 加上哈希映射部分的长度(HT_HASH_SIZE)得到到
#define HT_SET_DATA_ADDR(ht, ptr) do { \
        (ht)->arData = (Bucket*)(((char*)(ptr)) + HT_HASH_SIZE((ht)->nTableMask)); \
    } while (0)
```


然后是 `zend_hash_to_packed` ，这个初始化针对的是 哈希的键部分是数字的情况 这个时候并不需要 通过哈希值进行映射，所以这种情况的初始化并不会申请哈希映射部分的内存
```
ZEND_API void ZEND_FASTCALL zend_hash_to_packed(HashTable *ht)
{
    void *new_data, *old_data = HT_GET_DATA_ADDR(ht);
    Bucket *old_buckets = ht->arData;

    HT_ASSERT_RC1(ht);
    // 申请一块新的内存 注意这个内存的哈希映射部分是 HT_MIN_MASK 也就是只分配了两个
    new_data = pemalloc(HT_SIZE_EX(ht->nTableSize, HT_MIN_MASK), GC_FLAGS(ht) & IS_ARRAY_PERSISTENT);
    HT_FLAGS(ht) |= HASH_FLAG_PACKED | HASH_FLAG_STATIC_KEYS;
    // 掩码也设置成了最小值
    ht->nTableMask = HT_MIN_MASK;
    // 设置内存地址
    HT_SET_DATA_ADDR(ht, new_data);
    // 初始化哈希映射部分内存的值为 -1 
    HT_HASH_RESET_PACKED(ht);
    // 将数据拷贝到新分配的内存上
    memcpy(ht->arData, old_buckets, sizeof(Bucket) * ht->nNumUsed);
    // 释放老的内存
    pefree(old_data, GC_FLAGS(ht) & IS_ARRAY_PERSISTENT);
}
```

对于 packed 类型的 哈希表，php对其做了特殊处理，取消了哈希表的 哈希映射区域的内存。哈希的查询可以直接将 h 作为 元素的 索引进行操作。但是packed类型的哈希会很容易失效，数据索引一定要是顺序的，添加数据的时候索引一定要存在 不然就会被convert成hash类型的表
packed哈希表的查询
```
ZEND_API zval* ZEND_FASTCALL zend_hash_index_find(const HashTable *ht, zend_ulong h)
{
    Bucket *p;
    IS_CONSISTENT(ht);
    // 如果是 packed 类型
    if (HT_FLAGS(ht) & HASH_FLAG_PACKED) {
        // 查询 索引值一定是小于 nNumUsed 的
        if (h < ht->nNumUsed) {
            // 直接根据索引值 h 偏移
            p = ht->arData + h;
            // 值不空就返回了
            if (Z_TYPE(p->val) != IS_UNDEF) {
                return &p->val;
            }
        }
        return NULL;
    }
    // 当作 hash 类型的表查询了
    p = zend_hash_index_find_bucket(ht, h);
    return p ? &p->val : NULL;
}
```

然后是packed类型表的添加更新 看函数 `_zend_hash_index_add_or_update_i` 的部分截取
```
// packed 类型的哈希表
if (HT_FLAGS(ht) & HASH_FLAG_PACKED) {
    // 若 h 小于增长值 nNumUsed 则一定是更新操作
    if (h < ht->nNumUsed) {
        p = ht->arData + h;
        if (Z_TYPE(p->val) != IS_UNDEF) {
replace:
            // 要是添加操作直接返回 NULL 了
            if (flag & HASH_ADD) {
                return NULL;
            }
            if (ht->pDestructor) {
                ht->pDestructor(&p->val);
            }
            // 更新值
            ZVAL_COPY_VALUE(&p->val, pData);
            return &p->val;
        }
        // 若 h 对应的元素是空的 也就是说 不存在这个索引
        // 这个时候更新操作还要继续的话 packed 类型中是无法实现了 因为会破坏元素的顺序
        // 只能将 哈希表转成hash类型 然后走普通的更新操作
        else { /* we have to keep the order :( */
            goto convert_to_hash;
        }
    }
    // 索引 h 在范围 nNumUsed - nTableSize 则一定是添加操作
    else if (EXPECTED(h < ht->nTableSize)) {
add_to_packed:
        p = ht->arData + h;
        /* incremental initialization of empty Buckets */
        if ((flag & (HASH_ADD_NEW|HASH_ADD_NEXT)) != (HASH_ADD_NEW|HASH_ADD_NEXT)) {
            // 将 nNumUsed - h 部分的元素 置空
            // 这样在查询的时候 不存在的 idx 会返回 NULL
            if (h > ht->nNumUsed) {
                Bucket *q = ht->arData + ht->nNumUsed;
                while (q != p) {
                    ZVAL_UNDEF(&q->val);
                    q++;
                }
            }
        }
        ht->nNextFreeElement = ht->nNumUsed = h + 1;
        goto add;
    }
    // 哈希表可用元素过半 且 h 在 两倍 nTableSize 范围内 则进行扩容
    else if ((h >> 1) < ht->nTableSize &&
               (ht->nTableSize >> 1) < ht->nNumOfElements) {
        zend_hash_packed_grow(ht);
        goto add_to_packed;
    }
    // 转成hash类型
    else {
        if (ht->nNumUsed >= ht->nTableSize) {
            ht->nTableSize += ht->nTableSize;
        }
convert_to_hash:
        zend_hash_packed_to_hash(ht);
    }
}
// 未初始化 进行初始化处理
else if (HT_FLAGS(ht) & HASH_FLAG_UNINITIALIZED) {
    // 若索引在哈希表范围内 初始化成 packed表
    if (h < ht->nTableSize) {
        zend_hash_real_init_packed_ex(ht);
        goto add_to_packed;
    }
    // 转换成 hash类型表
    zend_hash_real_init_mixed(ht);
}
else {
    if ((flag & HASH_ADD_NEW) == 0) {
        p = zend_hash_index_find_bucket(ht, h);
        if (p) {
            goto replace;
        }
    }
    ZEND_HASH_IF_FULL_DO_RESIZE(ht);        /* If the Hash table is full, resize it */
}
```

#### 3.2 添加新元素

```
add_to_hash:
    // 紧邻可用的数据区内存索引
    idx = ht->nNumUsed++;
    ht->nNumOfElements++;
    arData = ht->arData;
    //p 是可以内存地址
    p = arData + idx;
    p->key = key;
    p->h = h = ZSTR_H(key);
    // 计算映射区的索引
    nIndex = h | ht->nTableMask;
    // 将映射区索引值赋值给 zval.u2.next
    Z_NEXT(p->val) = HT_HASH_EX(arData, nIndex);
    // 将数据区索引赋值给 映射区对应内存
    HT_HASH_EX(arData, nIndex) = HT_IDX_TO_HASH(idx);
    // 将pData数据拷贝到新的元素所在内存地址中
    ZVAL_COPY_VALUE(&p->val, pData);
```
需要解释的几个步骤
1. 计算可用数据区内存地址 
```
arData = ht->arData;
p = arData + idx;
```
ht->arData 的类型是 `Bucket * `，正对应数据区的数据类型 这个时候 + idx 其实是偏移量 偏移idx个 `Bucket *` 尺寸

2. 读取哈希映射区的值 这里使用的宏
```
#define HT_HASH_EX(data, idx) \
    ((uint32_t*)(data))[(int32_t)(idx)]
```
首先 idx 类型转成了 `int32_t` 这个时候是有符号的 也就是这个索引是负值
其次 data类型被转成 `uint32_t*` 一个是类型 `uint32_t` 对应哈希映射区的类型 一个是指针类型 这样可以用数组的形式读取内存数据

3. 数据拷贝 使用了宏 ZVAL_COPY_VALUE
```
#define ZVAL_COPY_VALUE(z, v)                           \
    do {                                                \
        zval *_z1 = (z);                                \
        const zval *_z2 = (v);                          \
        zend_refcounted *_gc = Z_COUNTED_P(_z2);        \
        uint32_t _t = Z_TYPE_INFO_P(_z2);               \
        ZVAL_COPY_VALUE_EX(_z1, _z2, _gc, _t);          \
    } while (0)

#define Z_COUNTED(zval)             (zval).value.counted
#define Z_COUNTED_P(zval_p)         Z_COUNTED(*(zval_p))

#define Z_TYPE_INFO(zval)           (zval).u1.type_info
#define Z_TYPE_INFO_P(zval_p)       Z_TYPE_INFO(*(zval_p))

# define ZVAL_COPY_VALUE_EX(z, v, gc, t)                \
    do {                                                \
        Z_COUNTED_P(z) = gc;                            \
        Z_TYPE_INFO_P(z) = t;                           \
    } while (0)
```
以上涉及到的宏能简化成如下结果
```
z.value.counted = v.value.counted
z.u1.type_info = v.u1.type_info
```
也就是zval的拷贝过程其实复制的就是 `zval.u1.type_info` 和 `zval.value.counted` 这两个值的属性，其中`zval.u1.type_info`存储的是变量类型信息，`zval.value.counted`存储的是变量的值。这里 `zval.value.counted` 其实是 `union zend_value`中的属性，作为联合体 这里读取的属性只提供的内存地址而已，其读取的值 取决于 最后一次对这块内存写入的值是多少，属性类型来看 都是指针类型的变量 占用内存长度都是 8 byte 。

哈希冲突链表的建立过程可以看下面这个演示图 :
![哈希冲突链表的建立过程](php-HashTable.gif)

#### 3.3 查找元素
熟悉哈希表的结构 元素的查找就好理解了，其逻辑过程如下：
1. 通过键字符串计算出 哈希值
2. 通过哈希值与掩码与运算计算出 哈希映射区的索引 并取出对应的冲突链表中的头节点
3. 取出头节点 对比  键字符串长度、 哈希值 和 对应的值 是否相等 相等则返回结束
4. 不想等则遍历冲突链表 通过 zval.u2.next 找到下一个元素 重复步骤 3 
5. 直到 zval.u2.next == HT_INVALID_IDX 结束遍历循环
下面是通过键字符串查找的代码例子
```
static zend_always_inline Bucket *zend_hash_str_find_bucket(const HashTable *ht, const char *str, size_t len, zend_ulong h)
{
    uint32_t nIndex;
    uint32_t idx;
    Bucket *p, *arData;

    arData = ht->arData;
    nIndex = h | ht->nTableMask;
    idx = HT_HASH_EX(arData, nIndex);
    // 遍历冲突链表 直到遇到 HT_INVALID_IDX
    while (idx != HT_INVALID_IDX) {
        ZEND_ASSERT(idx < HT_IDX_TO_HASH(ht->nTableSize));
        // 读取元素内存地址
        // 设置偏移量 到指定的 数据部分的内存地址
        p = HT_HASH_TO_BUCKET_EX(arData, idx);
        // 对比 哈希值、键值长度、值
        if ((p->h == h)
             && p->key
             && (ZSTR_LEN(p->key) == len)
             && !memcmp(ZSTR_VAL(p->key), str, len)) {
                // 找到则返回
            return p;
        }
        // 移动到冲突链表的下一个元素
        // #define Z_NEXT(zval)             (zval).u2.next
        idx = Z_NEXT(p->val);
    }
    return NULL;
}
```

## 4. 扩容
扩容部分使用了宏 
```
#define ZEND_HASH_IF_FULL_DO_RESIZE(ht)             \
    if ((ht)->nNumUsed >= (ht)->nTableSize) {       \
        zend_hash_do_resize(ht);                    \
    }
```
若 `nNumUsed >= nTableSize` 则通过函数 `zend_hash_do_resize` 进行扩容，也就是当添加次数 >= 哈希表尺寸的时候，这个条件不考虑删除的元素，删除的情况在函数 `zend_hash_do_resize` 中有处理，下面看下 函数 `zend_hash_do_resize`

```
static void ZEND_FASTCALL zend_hash_do_resize(HashTable *ht)
{

    IS_CONSISTENT(ht);
    HT_ASSERT_RC1(ht);

    // 这里考虑了删除元素的情况 达到一定阀值的时候 并不会直接进行两倍扩容 而是对哈希表中的元素进行收缩 清楚 删除元素
    if (ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)) { /* additional term is there to amortize the cost of compaction */
        zend_hash_rehash(ht);
    } else if (ht->nTableSize < HT_MAX_SIZE) {  /* Let's double the table size */
        // 两倍扩容
        void *new_data, *old_data = HT_GET_DATA_ADDR(ht);
        // 尺寸扩大到两倍 用加代理乘 提高计算效率
        uint32_t nSize = ht->nTableSize + ht->nTableSize;
        // 扩容前的数据
        Bucket *old_buckets = ht->arData;

        ht->nTableSize = nSize;
        // 申请新的扩容后的内存片段
        new_data = pemalloc(HT_SIZE_EX(nSize, HT_SIZE_TO_MASK(nSize)), GC_FLAGS(ht) & IS_ARRAY_PERSISTENT);
        // 重新计算哈希映射区
        ht->nTableMask = HT_SIZE_TO_MASK(ht->nTableSize);
        // 赋值新的内存地址
        HT_SET_DATA_ADDR(ht, new_data);
        // 将之前的数据拷贝到新申请的内存中
        memcpy(ht->arData, old_buckets, sizeof(Bucket) * ht->nNumUsed);
        // 释放之前内存段
        pefree(old_data, GC_FLAGS(ht) & IS_ARRAY_PERSISTENT);
        // 重新计算哈希映射区 以及数据区
        zend_hash_rehash(ht);
    } else {
        zend_error_noreturn(E_ERROR, "Possible integer overflow in memory allocation (%u * %zu + %zu)", ht->nTableSize * 2, sizeof(Bucket) + sizeof(uint32_t), sizeof(Bucket));
    }
}
```
注意这里并不是直接二倍扩容的，其分为两种情况，其中一个条件 `ht->nNumUsed > ht->nNumOfElements + (ht->nNumOfElements >> 5)` 并不会直接扩容，而是直接调整哈希 实现对删除元素的收缩。

删除的元素 超过哈希表尺寸的 1/33 ，也就是说 哈希表中 删除的元素占比超过 0.031 的时候 哈希表的扩容并不会启动 而是选择重建

然后是哈希重建的过程
```
ZEND_API int ZEND_FASTCALL zend_hash_rehash(HashTable *ht)
{
    Bucket *p;
    uint32_t nIndex, i;
    IS_CONSISTENT(ht);

    // hashtable 为空 初始化内存
    if (UNEXPECTED(ht->nNumOfElements == 0)) {
        if (!(HT_FLAGS(ht) & HASH_FLAG_UNINITIALIZED)) {
            ht->nNumUsed = 0;
            HT_HASH_RESET(ht);
        }
        return SUCCESS;
    }
    // 初始化 哈希映射部分的内存
    HT_HASH_RESET(ht);
    i = 0;
    p = ht->arData;
    // hashtable 中没有空洞 也就是没有被删除（type_info 设置为 UNDEF）的元素
    if (HT_IS_WITHOUT_HOLES(ht)) {
        do {
            // 上一步的宏 HT_HASH_RESET 会将 h与idx的映射段初始化 也就是之前的那段数据全部设置为 HT_INVALID_IDX 了
            // 所以这里其实是重新建立这段数据的关系 以及 bucket 链表关系
            nIndex = p->h | ht->nTableMask;
            Z_NEXT(p->val) = HT_HASH(ht, nIndex);
            HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(i);
            p++;
        } while (++i < ht->nNumUsed); // 循环遍历整个元素
    } else {
        // 这一步循环遍历数据部分 将删除的元素从哈希表中移除 然后将后面的数据向前移动
        do {
            // 如果当前元素是已经删除的
            if (UNEXPECTED(Z_TYPE(p->val) == IS_UNDEF)) {
                uint32_t j = i;
                Bucket *q = p;

                if (EXPECTED(!HT_HAS_ITERATORS(ht))) {
                    while (++i < ht->nNumUsed) {
                        p++;
                        // 此时 q 是后一个， p 是前一个 若 q == IS_UNDEF && p != IS_UNDEF 这时将 p 的数据复制到 q 中
                        if (EXPECTED(Z_TYPE_INFO(p->val) != IS_UNDEF)) {
                            ZVAL_COPY_VALUE(&q->val, &p->val);
                            q->h = p->h;
                            nIndex = q->h | ht->nTableMask;
                            q->key = p->key;
                            // 将q指向的元素添加到 冲突链表中
                            Z_NEXT(q->val) = HT_HASH(ht, nIndex);
                            HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(j);
                            if (UNEXPECTED(ht->nInternalPointer == i)) {
                                ht->nInternalPointer = j;
                            }
                            q++;
                            j++;
                        }
                    }
                } else {
                    // 迭代器的处理 代码逻辑类似上面步骤
                }
                ht->nNumUsed = j;
                break;
            }
            // 非删除元素直接调整
            nIndex = p->h | ht->nTableMask;
            Z_NEXT(p->val) = HT_HASH(ht, nIndex);
            HT_HASH(ht, nIndex) = HT_IDX_TO_HASH(i);
            p++;
        } while (++i < ht->nNumUsed); // 循环遍历整个元素
    }
    return SUCCESS;
}
```

这里对删除元素合并的操作代码单独分析下，将代码简化后如下:
```
do {
    // 当遇到删除元素 进行合并 将后面未删除元素拿来填充到前面的删除元素部分
    if (UNEXPECTED(Z_TYPE(p->val) == IS_UNDEF)) {
        uint32_t j = i;
        // 第一次遇到删除元素 此时 q p 均指向这个元素
        Bucket *q = p;
        // 基于第一个空元素往后遍历
        while (++i < ht->nNumUsed) {
            // p 往后遍历
            p++;
            // 直到在后面找到非删除的元素 将其拷贝到 q 指定的位置
            if (EXPECTED(Z_TYPE_INFO(p->val) != IS_UNDEF)) {
                // 拷贝数据 重建冲突链表
                // 这里并没有将 p 指向的非删除元素删除 因为没有必要
                // q是单步增长的 且不会判断是否为空 值是直接拷贝的
                
                // q 向前移动一个位置
                q++;
                j++;
            }
        }

        ht->nNumUsed = j;
        break;
    }

    // 重建冲突链表
    // ......
    p++;
} while (++i < ht->nNumUsed);
```
这个算法类似于删除数组中值为 0 的元素 且保持数组元素顺序
```
$arr = [21, 0, 34, 0, 0, 23, 11, 2, 0, 233, 89, 0, 0, 0, 783, 9, 0, 32, 0];
$len = count($arr);
$i = 0;

do {
    // 先找到第一个空洞
    if ($arr[$i] != 0) continue;
    // $i 指向列表第一个空洞
    $j = $i;
    while (++$j< $len) {
        // $j 指向后面第一个不为0的值
        if ($arr[$j] != 0) {
            // 将$j 的值 填充到 $i 
            $arr[$i] = $arr[$j];
            $i++;
        }
    }
    $arr = array_slice($arr, 0, $i);
    break;
}
while(++$i<$len);
```

## 5. 删除元素 

删除的过程先是查找到要删除的元素 `zend_hash_del` ，找到后执行 `_zend_hash_del_el_ex` 进行删除逻辑，删除的过程并不会真的从哈希表中移除，而是将要删除元素的值设置为 `IS_UNDEF` 类型。


哈希表删除元素的函数 这里主要逻辑是查询
```

ZEND_API int ZEND_FASTCALL zend_hash_del(HashTable *ht, zend_string *key)
{
    zend_ulong h;
    uint32_t nIndex;
    uint32_t idx;
    Bucket *p;
    Bucket *prev = NULL;

    IS_CONSISTENT(ht);
    HT_ASSERT_RC1(ht);
    // 通过key的hash code 获取数据存储的索引 idx
    h = zend_string_hash_val(key);
    nIndex = h | ht->nTableMask;
    idx = HT_HASH(ht, nIndex);

    // 在冲突链表中查询数据
    while (idx != HT_INVALID_IDX) {
        p = HT_HASH_TO_BUCKET(ht, idx);
        if ((p->key == key) ||
            (p->h == h &&
             p->key &&
             zend_string_equal_content(p->key, key))) {
            // 执行删除操作 这里会传递上一个元素的内存地址 prev 方便处理链表
            _zend_hash_del_el_ex(ht, idx, p, prev);
            return SUCCESS;
        }
        prev = p;
        idx = Z_NEXT(p->val);
    }
    return FAILURE;
}
```

执行真正的删除操作
```
static zend_always_inline void _zend_hash_del_el_ex(HashTable *ht, uint32_t idx, Bucket *p, Bucket *prev) {
    if (!(HT_FLAGS(ht) & HASH_FLAG_PACKED)) { // ht 不是纯数字索引
        // 将 p 从冲突链表中移除
        if (prev) { // p 不是第一个
            Z_NEXT(prev->val) = Z_NEXT(p->val);
        } else {
            // 头指针
            HT_HASH(ht, p->h | ht->nTableMask) = Z_NEXT(p->val);
        }
    }
    idx = HT_HASH_TO_IDX(idx);
    // nNumOfElements 记录哈希表中存储的元素总数
    ht->nNumOfElements--;

    // 这里是重置迭代
    if (ht->nInternalPointer == idx || UNEXPECTED(HT_HAS_ITERATORS(ht))) {
        // ......
    }
    // 删除的是最后一个元素
    if (ht->nNumUsed - 1 == idx) {
        do {
            // 此时这个值才会收缩
            ht->nNumUsed--;
        // 这里循环的条件 是 1. 还有元素 2. 从后往前遍历元素 如果碰到是 IS_UNDEF 的元素 这里会继续减 nNumUsed 否则就停止遍历
        } while (ht->nNumUsed > 0 && (UNEXPECTED(Z_TYPE(ht->arData[ht->nNumUsed-1].val) == IS_UNDEF)));
        ht->nInternalPointer = MIN(ht->nInternalPointer, ht->nNumUsed);
    }
    // 回收资源 这个元素要删除了
    if (p->key) {
        zend_string_release(p->key);
    }
    // 调用析构函数
    if (ht->pDestructor) {
        // 将这个要回收的变量传递给 析构函数
        zval tmp;
        ZVAL_COPY_VALUE(&tmp, &p->val);
        ZVAL_UNDEF(&p->val); // 这里在传递值之前先将他的 变量类型设置为 0
        ht->pDestructor(&tmp);
    } else {
        ZVAL_UNDEF(&p->val);
    }
}
```

删除的主要部分 一个是删除后需要在冲突链表中也将其删除 另一个就是设置其值为 IS_UNDEF 使用了宏 如下：
```
#define IS_UNDEF                    0
#define ZVAL_UNDEF(z) do {              \
        Z_TYPE_INFO_P(z) = IS_UNDEF;    \
    } while (0)
#define Z_TYPE_INFO_P(zval_p)       Z_TYPE_INFO(*(zval_p))
#define Z_TYPE_INFO(zval)           (zval).u1.type_info
// 展开的结果是 
zval.u1.type_info = 0;
```



















