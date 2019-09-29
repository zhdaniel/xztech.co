---
title: struct linger 用途介绍
tags:
  - tcp
  - linux
date: 2018-04-29 20:54:01
---

Linux下TCP连接端口的时候调用`close`函数，有优雅断开和强制断开两种，都是通过设置socket描述符的linger属性来实现。

linger数据结构如下：

````c
#include <arpa/inet.h>

struct linger {
    int l_onoff;
    int l_linger;
};
````

<!-- more -->

| 条件                         | 描述                                                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| l_onoff = 0                  | `close`立刻返回，底层会将未发送完的数据发送完之后再释放资源，即优雅退出。 | 内核默认。                                                   |
| l_onoff != 0 且 l_linger = 0 | `close`会立刻返回，但不会发送未发送完的数据，而是通过一个RST包强制的关闭socket描述符，即强制退出。 | TCP将丢弃保留在套接口发送缓冲区中的任何数据并发送一个RST给对方，   而不是通常的四分组终止序列，这避免了TIME_WAIT状态。 |
| l_onff != 0 且 l_linger > 0  | **`close`不会立刻返回，内核会延迟一段时间，这个时间就由 l_linger 的值来决定**。如果超时时间到达之前，发送完未发送的数据(包括FIN包)并得到另一端的确认，`close`会返回正确，socket描述符优雅性退出。否则，`close`会直接返回错误值，未发送数据丢失，socket描述符被强制性退出。需要注意的是，如果socket描述符被设置为非阻塞，则close()会直接返回值。 |                                                              |
|                              |                                                              |                                                              |

## 使用方法
```c
struct linger li = {0, 0};
setsockopt(socketfd, SOL_SOCKET, SO_LINGER, (void*)&li, sizeof(li));
```
