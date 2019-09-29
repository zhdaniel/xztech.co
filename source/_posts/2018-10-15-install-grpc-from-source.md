---
title: gRPC
tags:
  - grpc
date: 2018-10-15 22:58:02
---


编译安装gRPC（版本:1.14.2）
```shell
#!/bin/sh

DEST=$HOME/local/grpc-1.14.2
if [ -d $DEST ]; then
    /bin/rm -rf $DEST
fi
mkdir -p $DEST

# PKG_CONFIG_PATH  必须配置为绝对目录，不能是软连，否则配置无效
export PKG_CONFIG_PATH=$LOCAL/cares-1.11.0:$LOCAL/protobuf-3.6.0/lib/pkgconfig
export LDFLAGS=-L$LOCAL/protobuf/lib
export prefix=$DEST

make
make install

```
