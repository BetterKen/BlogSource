---
title: 4.MySQL体系结构
date: 2020-02-15 13:15:18
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---

## 4. 1 Mysql整体结构图

下图是Mysql官方给出的结构图:

![Mysql体系结构](http://mysql317.oss-cn-beijing.aliyuncs.com/Mysql%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84.png)



从上图可以发现Mysql由以下几部分组成

- 客户端
- 连接层
- SQL层
- 存储引擎层
- 文件系统
- 服务管理(非必须)



### 4.1.1 客户端

即Mysql提供给不同语言的API链接方式,支持的语言有很过Go,PHP,Java等等 没啥好说的哈



### 4.1.2 连接层

主要负责以下功能:

- 提供链接协议,两种: `socket连接&TCP/IP连接`
- 授权认证
- 提供专用的连接线程
- 最大连接数限制



### 4.1.3  SQL层

主要负责以下功能:

- SQL语法检查
- 语义检查(DML,DCL,DQL,DTL)
- 权限判断
- 解析器:解析预处理,执行计划(是全表扫描还是走哪个索引)
- 优化分析器:帮我们选择最优的方案
- 执行器:执行SQL语句
- 查询缓存(QC)
  - 【注】这个不要开启,在许多情况下它会失效并且会拉低数据库性能
  - 可以使用Redis来替代

### 4.1.4 存储引擎层

提供不同的存储引擎以供选择,类似于Linux的文件系统,和磁盘模块进行数据交互

我们可以通过show engine指令查看目前数据库支持的存储引擎

```
[root@centos7-1 3306]# mysql -e'show engines'|awk '{print $1}'
Engine
CSV
MRG_MYISAM
MyISAM        ***
BLACKHOLE
PERFORMANCE_SCHEMA
MEMORY          
ARCHIVE
InnoDB          *****
FEDERATED
```

经常提到的存储引擎有两个:

- InnoDB
- MyISAM

在早期使用的默认存储引擎为MyISAM,目前默认都是用InnoDB,他们之间的区别稍后会在存储引擎一篇中详细介绍



### 4.1.5 文件系统

用于存储数据文件和不同类型的日志



##  4.2 MySQL文件结构

目前我们的Mysql数据库下有以下几个库:

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| baseinfo           |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
6 rows in set (0.00 sec)
```

我们查看MySQL的数据存储文件夹下会看到如下文件:

```
[root@centos7-1 3306]# tree -L 1
.
├── auto.cnf
├── baseinfo
├── centos7-1.err
├── centos7-1.pid
├── ib_buffer_pool
├── ibdata1
├── ib_logfile0
├── ib_logfile1
├── ibtmp1
├── mysql
├── performance_schema
├── sys
└── test

```

意义如下:

```
[root@centos7-1 3306]# tree -L 1
.
├── auto.cnf   			#Mysql自动生成的一些配置文件
├── baseinfo            #baseinfo库里的文件【自建库】
├── centos7-1.err       #数据库错误日志 命名方式为 主机名+“.err”
├── centos7-1.pid       #数据里的进程id文件
├── ib_buffer_pool      #一些持久化了的buffer pool文件
├── ibdata1             #ibddata文件存储临时表数据+用户数据
├── ib_logfile0         #ib_logfile0～N为 redo log日志
├── ib_logfile1
├── ibtmp1              #临时表空间文件
├── mysql               #Mysql库文件
├── performance_schema  #performance_schema库文件夹【系统库】
├── sys		            #sys库文件夹【系统库】
└── test                #自建test库文件夹【自建库】

```



我们看到baseinfo库为Innodb存储引擎库,而test为MyISAM存储引擎库, 他们在文件存储上又有不同之处:

```
[root@centos7-1 3306]# tree test
test
├── db.opt
├── t1.frm
├── t1.MYD
└── t1.MYI

0 directories, 4 files
[root@centos7-1 3306]# tree baseinfo/
baseinfo/
├── db.opt
├── t_scrm_map.frm
├── t_scrm_map.ibd
├── t_scrm_pet_info.frm
├── t_scrm_pet_info.ibd
├── t_scrm_user_info.frm
└── t_scrm_user_info.ibd

[root@centos7-1 test]# cat db.opt 
default-character-set=utf8mb4
default-collation=utf8mb4_general_ci

#db.opt的作用
1、create database时会自动生成一个文件db.opt，存放的数据库的默认字符集，show create database时显示数据库默认字符集即db.opt中字符集
2、这个文件丢失不影响数据库运行，该文件丢失之后新建表时，找不到数据库的默认字符集，就把character_set_server当成数据库的默认字符集，show create database时显示character_set_server字符集
```



可以看到MyISAM库表的文件结构为三个:

```
${tablename}.frm   #表结构文件
${tablename}.MYI   #表索引文件 MyISAM Index
${tablename}.MYD   #表数据文件 MyISAM Data
```

而Innodb库表的文件结构为两个:

```
${tablename}.frm   #表结构文件
${tablename}.idb   #表索引及数据文件
```

可以看到在文件存储上 MyISAM和Innodb就有截然不同的结构 从而造成了数据引擎上的巨大差异



## 4.3  MySQL运行模式



MySQL被设计成一个单进程多线程架构的数据库,这一点与SQL Server比较类似,但与Oracle多进程的架构不同

`MySQL数据库实例在系统上表现就是一个进程`

已Innodb存储引擎为例,包含的后台线程主要有四种（具体含义会在存储引擎章节做解释）:

![](http://mysql317.oss-cn-beijing.aliyuncs.com/Mysqld%E8%BF%9B%E7%A8%8B.png)

我们可以在mysql中执行以下语句看查看所有mysql的进程数,具体含义在innodb章节解释吧:

```mysql
mysql> select thread_id,name,type FROM performance_schema.threads;
+-----------+----------------------------------------+------------+
| thread_id | name                                   | type       |
+-----------+----------------------------------------+------------+
|         1 | thread/sql/main                        | BACKGROUND |
|         2 | thread/sql/thread_timer_notifier       | BACKGROUND |
|         3 | thread/innodb/io_ibuf_thread           | BACKGROUND |
|         4 | thread/innodb/io_log_thread            | BACKGROUND |
|         5 | thread/innodb/io_read_thread           | BACKGROUND |
|         6 | thread/innodb/io_read_thread           | BACKGROUND |
|         7 | thread/innodb/io_read_thread           | BACKGROUND |
|         8 | thread/innodb/io_read_thread           | BACKGROUND |
|         9 | thread/innodb/io_write_thread          | BACKGROUND |
|        10 | thread/innodb/io_write_thread          | BACKGROUND |
|        11 | thread/innodb/io_write_thread          | BACKGROUND |
|        12 | thread/innodb/io_write_thread          | BACKGROUND |
|        13 | thread/innodb/page_cleaner_thread      | BACKGROUND |
|        16 | thread/innodb/srv_lock_timeout_thread  | BACKGROUND |
|        17 | thread/innodb/srv_error_monitor_thread | BACKGROUND |
|        18 | thread/innodb/srv_monitor_thread       | BACKGROUND |
|        19 | thread/innodb/srv_master_thread        | BACKGROUND |
|        20 | thread/innodb/srv_worker_thread        | BACKGROUND |
|        21 | thread/innodb/srv_purge_thread         | BACKGROUND |
|        22 | thread/innodb/srv_worker_thread        | BACKGROUND |
|        23 | thread/innodb/srv_worker_thread        | BACKGROUND |
|        24 | thread/innodb/buf_dump_thread          | BACKGROUND |
|        25 | thread/innodb/dict_stats_thread        | BACKGROUND |
|        26 | thread/sql/signal_handler              | BACKGROUND |
|        27 | thread/sql/compress_gtid_table         | FOREGROUND |
|        36 | thread/sql/one_connection              | FOREGROUND |
+-----------+----------------------------------------+------------+
26 rows in set (0.00 sec)

```



