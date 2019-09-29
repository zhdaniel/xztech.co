---
title: php internal - zend_string
tags:
  - php
  - zend
date: 2018-07-07 22:54:34
---

php7内部string实现

```c
typedef struct _zend_refcounted_h {
    uint32_t         refcount;            /* reference counter 32-bit */
    union {
        struct {
            ZEND_ENDIAN_LOHI_3(
                zend_uchar    type,
                zend_uchar    flags,    /* used for strings & objects */
                uint16_t      gc_info)  /* keeps GC root number (or 0) and color */
        } v;
        uint32_t type_info;
    } u;
} zend_refcounted_h;

typedef struct _zend_string     zend_string;

struct _zend_string {
    zend_refcounted_h gc;
    zend_ulong        h;             /* hash value */
    size_t            len;           /* 字符串长度 */
    char              val[1];        /* 并非长度并非1字节，便于使用且比使用 char * 减少一次内存分配 */
};
```
