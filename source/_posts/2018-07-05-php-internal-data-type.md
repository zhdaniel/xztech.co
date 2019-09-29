---
title: php internal - 数据类型
tags:
  - php
  - zend
date: 2018-07-05 22:36:02
---


# php internal - 数据类型

```c
/* regular data types */
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

/* constant expressions */
#define IS_CONSTANT                 11
#define IS_CONSTANT_AST             12

/* fake types */
#define _IS_BOOL                    13
#define IS_CALLABLE                 14
#define IS_ITERABLE                 19
#define IS_VOID                     18

/* internal types */
#define IS_INDIRECT                 15
#define IS_PTR                      17
#define _IS_ERROR                   20

/* we should never set just Z_TYPE, we should set Z_TYPE_INFO */
#define Z_TYPE(zval)                zval_get_type(&(zval))
#define Z_TYPE_P(zval_p)            Z_TYPE(*(zval_p))

#define Z_TYPE_FLAGS(zval)          (zval).u1.v.type_flags
#define Z_TYPE_FLAGS_P(zval_p)      Z_TYPE_FLAGS(*(zval_p))

#define Z_CONST_FLAGS(zval)         (zval).u1.v.const_flags
#define Z_CONST_FLAGS_P(zval_p)     Z_CONST_FLAGS(*(zval_p))

#define Z_TYPE_INFO(zval)           (zval).u1.type_info
#define Z_TYPE_INFO_P(zval_p)       Z_TYPE_INFO(*(zval_p))

static zend_always_inline zend_uchar zval_get_type(const zval* pz) {
	return pz->u1.v.type;
}

#define ZEND_SAME_FAKE_TYPE(faketype, realtype) ( \
	(faketype) == (realtype) \
	|| ((faketype) == _IS_BOOL && ((realtype) == IS_TRUE || (realtype) == IS_FALSE)) \
)
```
