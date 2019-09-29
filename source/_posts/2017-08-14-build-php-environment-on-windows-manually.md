---
title: Build PHP environment on Windows manually
date: 2017-08-14 00:28:31
tags:
    - php
---

### 环境需求
* PHP5.5.53
* MariaDB
* Apache2.4

### 安装MariaDB
1. 下载32或64的[MariaDB]，选择ZIP file即可
2. 解压Zip包到指定目录
3. 将`my.ini`修改
4. 进入`bin`目录，执行`mysqld --install MySQL --defaults-file="D:\Tools\mariadb-10.1.9-winx64\my.ini"`
5. 至此MySQL已安装完成，若有错误可以`WIN + R eventvwr`查看详情

<!-- more -->

### 安装PHP
1. 下载32或者64的[PHP]，由于本环境使用Apache所以下载的选择线程安全的版本，若使用IIS则选择非线程安全的版本
2. 解压下载的压缩包，将该目录添加至系统环境变量，如：新建PHP变量，值为`D:\Tools\php-5.5.33-Win32-VC11-x64`，再将PHP变量添加至PATH
3. 至此PHP已安装完成

### 安装Apache
1. 下载32或64位的[Apache]
2. 解压下载的压缩包，进入`conf`目录编辑`httpd.conf`文件
3. 新增PHP模块加载：`LoadModule php5_module "D:/Tools/php-5.5.33-Win32-VC11-x64/php5apache2_4.dll"`
4. 编辑`DocumentRoot`指定网站根目录如：`D:/Tools/httpd-2.4.18-win64-VC14/htdocs`
5. 增加php后缀的默认文件，在`<IfModule dir_module>`中增加`index.php`
6. 在`<IfModule mime_module>`节点中增加PHP处理器：`AddHandler application/x-httpd-php .php`
7. 在配置文件最后加入PHP执行文件所在路径：`PHPIniDir "D:/Tools/php-5.5.33-Win32-VC11-x64"`
8. 进入`bin`目录，执行`httpd -k install`安装Apache为Windows服务（安装时可能需要管理员权限）
9. 至此Apache已安装完成，若出现错误可以`WIN + R eventvwr`查看详情

[MariaDB]:https://downloads.mariadb.org/
[PHP]:http://windows.php.net/download/
[Apache]:https://www.apachelounge.com/
