---
date: 2022-09-09T22:52:30+08:00  # 创建日期
author: "Rustle Karl"  # 作者

title: "Redis 底层数据结构 SDS"  # 文章标题
url:  "posts/redis/docs/internal/sds"  # 设置网页永久链接
tags: [ "redis", "sds" ]  # 标签
categories: [ "Redis 学习笔记" ]  # 分类

toc: true  # 目录
draft: true  # 草稿
---

SDS(`simple dynamic string`)，这是一种用于**存储二进制数据**的一种结构，具有**动态扩容**的特点。

其实现位于 `src/sds.h` 与 `src/sds.c`中，其关键定义如下：

```c
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

SDS 的总体概览如下图：

![](https://dd-static.jd.com/ddimg/jfs/t1/139197/15/29303/2064/631b3366E509243cb/491b2e233f16e438.png)

其中 sdshdr 是头部，buf 是真实存储用户数据的地方。另外注意，从命名上能看出来，这个数据结构除了能存储二进制数据，显然是用于设计作为字符串使用的，所以在 buf 中，用户数据后总跟着一个 \0。即图中 " 数据 " + "\0" 是为所谓的 buf。

SDS 有五种不同的头部。其中 sdshdr5 实际并未使用到。所以实际上有四种不同的头部，分别如下：

![](https://dd-static.jd.com/ddimg/jfs/t1/145236/30/30963/7663/631b3369E787611ba/57563536b7db24bd.png)


- len 分别以 uint8，uint16，uint32，uint64 表示用户数据的长度 ( 不包括末尾的 \0) 
- alloc 分别以 uint8，uint16，uint32，uint64 表示整个 SDS，除过头部与末尾的 \0，剩余的字节数。
- flag 始终为一字节，以低三位标示着头部的类型，高 5 位未使用。

当在程序中持有一个 SDS 实例时，直接持有的是数据区的头指针，这样做的用意是：通过这个指针，向前偏一个字节，就能取到 flag，通过判断 flag 低三位的值，能迅速判断：头部的类型，已用字节数，总字节数，剩余字节数。这也是为什么 sds 类型即是 char * 指针类型别名的原因。

创建一个 SDS 实例有三个接口，分别是：

```c
// 创建一个不含数据的sds: 
//  头部    3字节 sdshdr8
//  数据区  0字节
//  末尾    \0 占一字节
sds sdsempty(void);
// 带数据创建一个sds:
//  头部    按initlen的值, 选择最小的头部类型
//  数据区  从入参指针init处开始, 拷贝initlen个字节
//  末尾    \0 占一字节
sds sdsnewlen(const void *init, size_t initlen);
// 带数据创建一个sds:
//  头部    按strlen(init)的值, 选择最小的头部类型
//  数据区  入参指向的字符串中的所有字符, 不包括末尾 \0
//  末尾    \0 占一字节
sds sdsnew(const char *init);
```

- 所有创建 sds 实例的接口，都不会额外分配预留内存空间

- `sdsnewlen` 用于带二进制数据创建 sds 实例，sdsnew 用于带字符串创建 sds 实例。接口返回的 sds 可以直接传入 libc 中的字符串输出函数中进行操作，由于无论其中存储的是用户的二进制数据，还是字符串，其末尾都带一个 \0，所以至少调用 libc 中的字符串输出函数是安全的。

在对 SDS 中的数据进行修改时，若剩余空间不足，会调用 sdsMakeRoomFor 函数用于扩容空间，这是一个很低级的 API，通常情况下不应当由 SDS 的使用者直接调用。其实现中核心的几行如下：

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    ...
    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;
    ...
}
```

可以看到，在扩充空间时：

- 先保证至少有 addlen 可用

- 然后再进一步扩充，在总体占用空间不超过阈值 `SDS_MAC_PREALLOC` 时，申请空间再翻一倍。若总体空间已经超过了阈值，则步进增长 `SDS_MAC_PREALLOC`。这个阈值的默认值为 `1024 * 1024`

SDS 也提供了接口用于移除所有未使用的内存空间。`sdsRemoveFreeSpace`，该接口没有间接的被任何 SDS 其它接口调用，即默认情况下，SDS 不会自动回收预留空间。在 SDS 的使用者需要节省内存时，由使用者自行调用：

```c
sds sdsRemoveFreeSpace(sds s);
```
