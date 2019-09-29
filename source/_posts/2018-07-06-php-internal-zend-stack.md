---
title: php internal - zend_stack
tags:
  - php
  - zend
date: 2018-07-06 22:50:47
---


- **zend_stack 定义**

```c

typedef struct _zend_stack {
// size为栈中最大元素的大小
	int size, top, max;
	void *elements;
} zend_stack;

#define ZEND_STACK_ELEMENT(stack, n) ((void *)((char *) (stack)->elements + (stack)->size * (n)))

+--------+            +- size -+
|size    |            +--------+
|top     |     +----->|        |
|max     |     |      |        |
|elments +-----+      |        |
+--------+            |        |
                      +--------+
```

<!-- more -->

- **栈的常用API**

```c
ZEND_API int zend_stack_init(zend_stack *stack, int size);
ZEND_API int zend_stack_push(zend_stack *stack, const void *element);
ZEND_API void *zend_stack_top(const zend_stack *stack);
ZEND_API int zend_stack_del_top(zend_stack *stack);
ZEND_API int zend_stack_int_top(const zend_stack *stack);
ZEND_API int zend_stack_is_empty(const zend_stack *stack);
ZEND_API int zend_stack_destroy(zend_stack *stack);
ZEND_API void *zend_stack_base(const zend_stack *stack);
ZEND_API int zend_stack_count(const zend_stack *stack);
ZEND_API void zend_stack_apply(zend_stack *stack, int type, int (*apply_function)(void *element));
ZEND_API void zend_stack_apply_with_argument(zend_stack *stack, int type, int (*apply_function)(void *element, void *arg), void *arg);
ZEND_API void zend_stack_clean(zend_stack *stack, void (*func)(void *), zend_bool free_elements);
```

- **zend_stack的应用**
  - 词法分析
  - 注册错误和异常处理函数
