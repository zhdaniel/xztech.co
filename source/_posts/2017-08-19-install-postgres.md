---
title: 解压安装PostgreSQL
tags:
  - posgresql
  - database
date: 2017-08-19 20:47:33
---

最近工作需要用到GPDB，考虑到GPDB是基于PostgreSQL开发且协议也和后者完全兼容，所以先在Windows上搭建一个简单的环境学习下PostgreSQL。

+ 下载PostgreSQL
+ 解压到指定目录，如：D:\Tools\pgsql
+ 指定数据库文件目录，增加环境变量 PGDATA=D:\Tools\pgsql\data，后面的相关操作就无需显示指定数据文件目录
+ 初始化数据库目录：initdb.exe -E UTF8 --locale C
+ 将PostgreSQL注册Windows服务：pg_ctl.exe register
+ 启动服务
