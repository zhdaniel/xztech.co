---
title: GCC中强符号和弱符号
tags:
  - gcc
  - c
  - c++
date: 2019-12-01 18:52:40
---

## 背景

全局变量虽然非常强大但是使用的时候也有很大的风险。在大多数情况下，我们可以在全局变量上添加 `static` 修饰符，这样这些变量就只能在所在的文件中被修改。然而，有时候我们也存在跨不同文件使用全局变量的情况，这时我们一般会在链接阶段遇到一些错误，如：`error: multiple definition`。

这里有如下两个源文件：

```c
#include <stdio.h>

// main.c
int g_var;

int main(int argc, char* argv[])
{
    printf("global var is %d\n", g_var);
    return 0;
}

// config.c

int g_var = 5;
```

<!-- more -->

然后我们执行编译：

```shell
gcc -o main main.c config.c
```

没有报任何错误，很顺利的编译通过了，变量 `g_var` 也没有报 `multiple definition` 。变量 `g_var` 前没有 `extern` 修饰符，执行结果 `g_var` 为 `5` ，这也以为 `g_var` 跨两个文件共享了。看似很奇怪，因为我们知道有多个相同名的全局变量会报 `multiple definition` 。



另外非常有趣的是，如果更改这两个文件的后缀，则会报错：

```shell
mv main.c main.cpp
mv config.c config.cpp
gcc -o main main.cpp config.cpp
# error: multiple definition
```



## 强符号和弱符号

实际上这些现象是GCC的强符号和弱符号功能导致的，对于全局变量，可以分为三种情况：

+ 初始化非零值
+ 初始化零值
+ 未初始化

在GCC中，前两种情况的全局变量成为强符号，并分别存在 `.DATA` 和 `.BSS` 段中，第三种情况成为弱符号，它保存在 `.COMMON` 段中。



这些变量必须遵循如下三条规则：

+ 同名强符号最多只能存在一个
+ 如果存在一个强符号和多个若符号，则弱符号会被强符号覆盖
+ 如果存在多个弱符号，GCC会选择其中内存最大的一个

所以，之所以在上面的例子中C语言可以正常运行，在 `config.c` 文件中我们定义了一个强符号 `g_var` 并将它初始化为 `5` 。在 `main.c` 中只是声明一个变量并未赋值，它是弱符号。当我们使用GCC编译的时候，根据上面的第二个规则，`main.c` 中的 `g_var` 会被 `config.c` 中的覆盖。但是为什么将文件名后缀修改后再编译会报错呢？那是因为在C++中，默认为强符号，所以如果要在C++版本正常运行则需要显示声明为弱符号。

```c++
#include <stdio.h>

// main.cpp
int __attribute__((weak)) g_var = 1;

int main()
{
    printf("shared var is %d\n", g_var);
    return 0;
}
```



为了避免这种情况导致bug，在编译的时候可以加上 `-fno-common` 参数，编译器会将所有变量都当作强符号。然后有时候我们不可避免使用弱符号，所以建议有一个良好的编码习惯，如下面三条规则：

+ 不使用全局变量（很难）
+ 为全局变量增加 `static` 关键字，通过提供的接口来访问（中等）
+ 初始化全局变量（容易）

