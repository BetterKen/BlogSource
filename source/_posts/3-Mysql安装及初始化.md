---
title: 3.Mysql安装及初始化
date: 2020-02-14 13:15:17
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---



环境:Centos8 64位
版本:5.7.26 通用二级制版本
下载地址:   https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz

## 3.1 下载通用二进制包

```shell
[root@localhost Downloads]# wget  https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.26-linux-glibc2.12-x86_64.tar.gz
```

## 3.2 创建用户用户
```
[root@localhost ken]# useradd mysql
```
## 3.3 创建相应文件夹

```
#创建数据存放目录
[root@localhost ken]# mkdir -p /data/3306
#创建mysql程序存放的目录
[root@localhost ken]# mkdir -p /app/database
#创建日志存放的目录
[root@localhost ken]# mkdir -p /log/3306
```

## 3.4 更改所属目录
```
[root@localhost ken]# chown -R mysql.mysql /data/3306  /app/database/ /log/3306/
```

## 3.5 解压mysql并放到指定目录

```
[ken@localhost Downloads]$ tar -zxvf mysql-5.7.26-linux-glibc2.12-i686.tar.gz 
[ken@localhost Downloads]$ ls
mysql-5.7.26-linux-glibc2.12-i686  mysql-5.7.26-linux-glibc2.12-i686.tar.gz
[ken@localhost Downloads]$ sudo mv mysql-5.7.26-linux-glibc2.12-i686 /app/database/mysql
[ken@localhost Downloads]$ sudo chown -R mysql.mysql /app/database/mysql
```
## 3.6设置环境变量
```
[ken@localhost ~]$ su
Password: 
[root@localhost ken]# echo "export PATH=/app/database/mysql/bin:$PATH" >> /etc/profile
[root@localhost ken]# exit
exit
[ken@localhost ~]$ source /etc/profile
#显示以下内容即可完成此安装步骤
[ken@localhost bin]$ mysql -V
mysql  Ver 14.14 Distrib 5.7.26, for linux-glibc2.12 (x86_64) using  EditLine wrapper

【注意】在centos8中 经常会报libncurses.so.5这个文件不存在 但是已经安装了libncurses库还是有报错的话
主要是再ceontos中有它的升级版文件/lib64/libncurses.so.6.1 我们做一个软连接即可
ln -s /lib64/libncurses.so.6.1 /lib64/libncurses.so.5
```

## 3.7 初始化Mysql

```
[root@localhost ken]# mysqld --initialize-insecure --user=mysql --basedir=/app/database/mysql --datadir=/data/3306
```

## 3.8 配置文件设置

```
cat > /etc/my.cnf <<EOF
[mysqld]
user=mysql
basedir=/app/database/mysql
datadir=/data/3306
socket=/tmp/mysql3306.sock
[mysql]
socket=/tmp/mysql3306.sock
EOF
```

## 3.9 准备Mysql启动脚本

```
root@localhost support-files]# pwd
/app/database/mysql/support-files
[root@localhost support-files]# cp mysql.server /etc/init.d/mysqld
[root@localhost support-files]# service mysqld start
Starting MySQL. SUCCESS! 

```
## 3.10 验证是否可以连接成功

```
[root@localhost support-files]# mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.26 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```


