---
title: 11.Mysql日志
date: 2020-02-18 20:16:00
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---


主要包括:

1. 错误日志 Error Log
2. 慢查询日志 Slow Query Log
3. 二进制日志 Bin Log
4. 重做日志 Redo log
5. 回滚日志 Undo Log
6. 中继日志 Relay Log
7. 一般查询日志 Gerneral Log



## 11.1 Error Log



### 11.1.1 作用 

用来记录从服务器启动和关闭过程中的信息（未必是错误信息，如mysql如何启动InnoDB的表空间文件的、如何初始化自己的存储引擎的等等）、服务器运行过程中的错误信息、事件调度器运行一个事件时产生的信息、在从服务器上启动服务器进程时产生的信息

在mysql数据库中，错误日志功能是默认开启的

默认情况下，错误日志存储在mysql数据库的数据文件中。

错误日志文件通常的名称为hostname.err。其中，hostname表示服务器主机名。



### 11.1.2 相关参数

```mysql
#错误日志存储的位置以及名称
mysql> select @@log_error;
+-----------------+
| @@log_error     |
+-----------------+
| ./centos7-1.err |
+-----------------+
1 row in set (0.00 sec)

#错误级别
mysql> select @@log_error_verbosity;
+-----------------------+
| @@log_error_verbosity |
+-----------------------+
|                     3 |
+-----------------------+
1 row in set (0.00 sec)

#错误级别　已废弃不推荐使用的　等同于log_error_verbosity
mysql> select @@log_warnings;
+----------------+
| @@log_warnings |
+----------------+
|              2 |
+----------------+
1 row in set, 1 warning (0.00 sec)


```

### 11.1.3 修改记录级别

![](http://mysql317.oss-cn-beijing.aliyuncs.com/error_verbosity.png)



### 11.1.4 清空归档日志

```shell
mv host_name.err host_name.err-old
mysqladmin flush-logs
mv host_name.err-old backup-directory
```



## 11.2 Slow Query Log

### 11.2.1 作用

查询日志是用来记录执行时间超过指定时间的查询语句。通过慢查询日志，可以查找出哪些查询语句的执行效率很低，以便进行优化。一般建议开启，它对服务器性能的影响微乎其微，但是可以记录mysql服务器上执行了很长时间的查询语句。可以帮助我们定位性能问题

### 11.2.2 相关参数

```mysql
#是否开启慢查询 1开启 0关闭
mysql> select @@slow_query_log;
+------------------+
| @@slow_query_log |
+------------------+
|                1 |
+------------------+
1 row in set (0.00 sec)

# 慢查询文件位置
mysql> select @@slow_query_log_file;
+----------------------------+
| @@slow_query_log_file      |
+----------------------------+
| /log/3306/slowlog/slow.log |
+----------------------------+
1 row in set (0.00 sec)

# 慢查询时间阀值　单位为秒
mysql> select @@long_query_time;
+-------------------+
| @@long_query_time |
+-------------------+
|          5.000000 |
+-------------------+
1 row in set (0.00 sec)

# 1开启　０关闭
# 运行的SQL语句没有使用索引，则MySQL数据库同样会将这条SQL语句记录到慢查询日志文件
mysql> select @@log_queries_not_using_indexes;
+---------------------------------+
| @@log_queries_not_using_indexes |
+---------------------------------+
|                               1 |
+---------------------------------+
1 row in set (0.00 sec)

```



### 11.2.3 查看slow.log

slow.log属于文本日志,我们可以通过linux命令查看slow.log

```shell
[root@centos7-1 slowlog]# tail -f slow.log 
# Time: 2019-07-09T02:00:48.300499Z
# User@Host: root[root] @ localhost []  Id:     5
# Query_time: 5.075423  Lock_time: 0.000123 Rows_sent: 174  Rows_examined: 4994309
SET timestamp=1562637648;
select * from t_scrm_pet_info where pet_name ="Winter";
```

含义如下:

- SQL 的执行时间：# Time: 2019-07-09T02:00:48.300499Z
- SQL 的执行主机：# User@Host: root[root] @ localhost []  Id:     5
- SQL 的执行信息：# Query_time: 5.075423  Lock_time: 0.000123 Rows_sent: 174  Rows_examined: 4994309
- SQL 的执行时间：SET timestamp=1562637648;
- SQL 的执行内容：select * from t_scrm_pet_info where pet_name ="Winter";

### 11.2.4 mysqldumpslow

mysqldumpslow 是一个针对于 MySQL 慢查询的命令行程序。可以通过 mysqldumpslow 查找出查询较慢的 SQL 语句。

```shell
[root@centos7-1 slowlog]# mysqldumpslow --help
Usage: mysqldumpslow [ OPTS... ] [ LOGS... ]

Parse and summarize the MySQL slow query log. Options are

  --verbose    verbose
  --debug      debug
  --help       write this text to standard output

  -v           verbose
  -d           debug
  -s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default
                al: average lock time
                ar: average rows sent
                at: average query time
                 c: count
                 l: lock time
                 r: rows sent
                 t: query time  
  -r           reverse the sort order (largest last instead of first)
  -t NUM       just show the top n queries
  -a           don't abstract all numbers to N and strings to 'S'
  -n NUM       abstract numbers with at least n digits within names
  -g PATTERN   grep: only consider stmts that include this string
  -h HOSTNAME  hostname of db server for *-slow.log filename (can be wildcard),
               default is '*', i.e. match all
  -i NAME      name of server instance (if using mysql.server startup script)
  -l           don't subtract lock time from total time

```

通过mysqldumpslow  查看慢日志：

```shell
[root@centos7-1 slowlog]# mysqldumpslow slow.log 

Reading mysql slow query log from slow.log
Count: 3  Time=5.12s (15s)  Lock=0.00s (0s)  Rows=1661.3 (4984), root[root]@localhost
  select * from t_scrm_pet_info where pet_name ="S"
```

- Count：出现次数,
- Time：执行最长时间和累计总耗费时间
- Lock：等待锁的时间
- Rows：返回客户端行总数和扫描行总数



## 11.3 Binlog 

### 11.3.1 作用

主要记录数据库变化(DDL,DCL,DML)性质的日志,是逻辑层性质日志

主要有以下作用:

- 恢复(recovery)： 对于误删除的数据可以通过binlog恢复
- 复制(replication) :可以通过binlog做主从同步
- 审计(audit)：可以通过抓取binlog日志来满足审计需求

### 11.3.2 相关参数

```ini
cat /etc/my.cnf
....
#主机编号　主从同步的时候使用　开启binlog需要加此参数
server_id=1            
#日志存放的目录＋日志名前缀 :mysql-bin.000001 mysql-bin.000002 ...
log_bin=/log/3306/binlog/mysql-bin
#binlog日志刷盘策略【重要】
sync_binlog=1
#binlig的记录格式
binlog_format=row
#指定单个binlog文件的大小(默认1G)
max_binlog_size=200M

```

#### 11.3.2.1 binglog_format

二进制日志有以下几种格式可选:

- STATEMENT模式（SBR）

  每一条会修改数据的sql语句会记录到binlog中。优点是并不需要记录每一条sql语句和每一行的数据变化，减少了binlog日志量，节约IO，提高性能。缺点是在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last_insert_id()，以及user-defined functions(udf)等会出现问题

- ROW模式（RBR）(`5.7版本默认`)

  不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了，修改成什么样了。而且不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题。缺点是会产生大量的日志，尤其是alter table的时候会让日志暴涨。

- MIXED模式（MBR）

  以上两种模式的混合使用，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式。



| 模式 | 优点                                 |   缺点   |
| :--: | :----------------------------------- | :------: |
| SBR  | 可读性高,日志量少,主从版本可以不一样 | 不够严禁 |
| RBR  | 可读性底,日志量大,主从版本需要统一   | 足够严禁 |



## 11.4 Redo Log

## 11.5 Undo Log







