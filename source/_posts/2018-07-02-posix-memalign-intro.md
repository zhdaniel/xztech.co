---
title: posix_memalign 函数
tags:
  - linux
  - posix
date: 2018-07-02 22:25:53
---

**posix_memalign**

+ 功能：函数`posix_memalign()`分配size字节大小内存，并将分配的内存地址放入`*memptr`中。分配的内存地址是`alignment`的倍数，且该地址一定是`sizeof(void*)`的偶数倍（2倍）。如果`size = 0`那么`posix_memalign`返回`NULL`或一个指针以便传递给`free`函数。

<!-- more -->

```c
#include <stdlib.h>

// posix_memalign is available since glibc 2.1.91
int posix_memalign(void **memptr, size_t alignment, size_t size);

// aligned_alloc is available since glibc 2.16, C11标准
// aligned_alloc 函数除了增加了：size 必须是 alignment 的倍数，其余和 memalign 一样
void *aligned_alloc(size_t alignment, size_t size);

// valloc: OBSOLETE
// 分配 size 字节大小内存并返回该内存地址
// 该内存地址是页大小的倍数，等价：memalign(sysconf(_SC_PAGESIZE), size)
void *valloc(size_t size);

#include <malloc.h>
// memalign: OBSOLETE
// memalign 功能和 posix_memalign一样，但已被废弃
void *memalign(size_t alignment, size_t size);
// pvalloc: OBSOLETE
// pvalloc 和 valloc 功能类似，但是对分配内存大小根据页大小做了舍入处理
void *pvalloc(size_t size);

// glibc 功能测试宏
posix_memalign():
	_POSIX_C_SOURCE >= 200112L || _XOPEN_SOURCE >= 600

aligned_alloc():
	_ISOC11_SOURCE

valloc():
// Since glibc 2.12:
_BSD_SOURCE ||
    (_XOPEN_SOURCE >= 500 ||
        _XOPEN_SOURCE && _XOPEN_SOURCE_EXTENDED) &&
    !(_POSIX_C_SOURCE >= 200112L || _XOPEN_SOURCE >= 600)

// Before glibc 2.12:
_BSD_SOURCE || _XOPEN_SOURCE >= 500 || _XOPEN_SOURCE && _XOPEN_SOURCE_EXTENDED
```
