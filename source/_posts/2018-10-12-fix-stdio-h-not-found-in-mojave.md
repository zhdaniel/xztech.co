---
title: Mojave 中找不到 stdio.h 解决方案
tags:
  - macosx
date: 2018-10-12 22:32:24
---

最近开发机升级到Mojave之后，编译PHP的时候报错提示无法找到 stdio.h，解决方案如下：
```shell
xcode-select --install
# OR
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```
