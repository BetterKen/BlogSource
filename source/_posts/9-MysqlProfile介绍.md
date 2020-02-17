---
title: 9.MysqlProfile介绍
date: 2020-02-16 19:18:10
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---

## 9.1　相关参数

```mysql
mysql> show variables like '%profil%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| have_profiling         | YES   |
| profiling              | OFF   |
| profiling_history_size | 15    |
+------------------------+-------+
3 rows in set (0.01 sec)


have_profiling： 当前版本是否支持profiling功能
profiling： 是否开启profiling功能
profiling_history_size： 保留profiling的数目，默认是15，范围为0~100，为0时代表禁用profiling

```



## 9.2 开启profile



```mysql
session级别开启:
mysql> SET profiling = 1;
全局开启
echo "profiling=1" >> my.cnf
```



## 9.3 help profile

我们可以通过help profile查看profile的用法,官方的解释已经很好了

```mysql
mysql> help profile;
Name: 'SHOW PROFILE'
Description:
Syntax:
SHOW PROFILE [type [, type] ... ]
    [FOR QUERY n]
    [LIMIT row_count [OFFSET offset]]

type: {
    ALL
  | BLOCK IO
  | CONTEXT SWITCHES
  | CPU
  | IPC
  | MEMORY
  | PAGE FAULTS
  | SOURCE
  | SWAPS
}

The SHOW PROFILE and SHOW PROFILES statements display profiling
information that indicates resource usage for statements executed
during the course of the current session.

*Note*:

The SHOW PROFILE and SHOW PROFILES statements are deprecated and will
be removed in a future MySQL release. Use the Performance Schema
instead; see
http://dev.mysql.com/doc/refman/5.7/en/performance-schema-query-profili
ng.html.

To control profiling, use the profiling session variable, which has a
default value of 0 (OFF). Enable profiling by setting profiling to 1 or
ON:

mysql> SET profiling = 1;

SHOW PROFILES displays a list of the most recent statements sent to the
server. The size of the list is controlled by the
profiling_history_size session variable, which has a default value of
15. The maximum value is 100. Setting the value to 0 has the practical
effect of disabling profiling.

All statements are profiled except SHOW PROFILE and SHOW PROFILES, so
you will find neither of those statements in the profile list.
Malformed statements are profiled. For example, SHOW PROFILING is an
illegal statement, and a syntax error occurs if you try to execute it,
but it will show up in the profiling list.

SHOW PROFILE displays detailed information about a single statement.
Without the FOR QUERY n clause, the output pertains to the most
recently executed statement. If FOR QUERY n is included, SHOW PROFILE
displays information for statement n. The values of n correspond to the
Query_ID values displayed by SHOW PROFILES.

The LIMIT row_count clause may be given to limit the output to
row_count rows. If LIMIT is given, OFFSET offset may be added to begin
the output offset rows into the full set of rows.

By default, SHOW PROFILE displays Status and Duration columns. The
Status values are like the State values displayed by SHOW PROCESSLIST,
although there might be some minor differences in interpretion for the
two statements for some status values (see
http://dev.mysql.com/doc/refman/5.7/en/thread-information.html).

Optional type values may be specified to display specific additional
types of information:

o ALL displays all information

o BLOCK IO displays counts for block input and output operations

o CONTEXT SWITCHES displays counts for voluntary and involuntary
  context switches

o CPU displays user and system CPU usage times

o IPC displays counts for messages sent and received

o MEMORY is not currently implemented

o PAGE FAULTS displays counts for major and minor page faults

o SOURCE displays the names of functions from the source code, together
  with the name and line number of the file in which the function
  occurs

o SWAPS displays swap counts

Profiling is enabled per session. When a session ends, its profiling
information is lost.

URL: http://dev.mysql.com/doc/refman/5.7/en/show-profile.html

Examples:
mysql> SELECT @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set (0.00 sec)

mysql> SET profiling = 1;
Query OK, 0 rows affected (0.00 sec)

mysql> DROP TABLE IF EXISTS t1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> CREATE TABLE T1 (id INT);
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW PROFILES;
+----------+----------+--------------------------+
| Query_ID | Duration | Query                    |
+----------+----------+--------------------------+
|        0 | 0.000088 | SET PROFILING = 1        |
|        1 | 0.000136 | DROP TABLE IF EXISTS t1  |
|        2 | 0.011947 | CREATE TABLE t1 (id INT) |
+----------+----------+--------------------------+
3 rows in set (0.00 sec)

mysql> SHOW PROFILE;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| checking permissions | 0.000040 |
| creating table       | 0.000056 |
| After create         | 0.011363 |
| query end            | 0.000375 |
| freeing items        | 0.000089 |
| logging slow query   | 0.000019 |
| cleaning up          | 0.000005 |
+----------------------+----------+
7 rows in set (0.00 sec)

mysql> SHOW PROFILE FOR QUERY 1;
+--------------------+----------+
| Status             | Duration |
+--------------------+----------+
| query end          | 0.000107 |
| freeing items      | 0.000008 |
| logging slow query | 0.000015 |
| cleaning up        | 0.000006 |
+--------------------+----------+
4 rows in set (0.00 sec)

mysql> SHOW PROFILE CPU FOR QUERY 2;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| checking permissions | 0.000040 | 0.000038 |   0.000002 |
| creating table       | 0.000056 | 0.000028 |   0.000028 |
| After create         | 0.011363 | 0.000217 |   0.001571 |
| query end            | 0.000375 | 0.000013 |   0.000028 |
| freeing items        | 0.000089 | 0.000010 |   0.000014 |
| logging slow query   | 0.000019 | 0.000009 |   0.000010 |
| cleaning up          | 0.000005 | 0.000003 |   0.000002 |
+----------------------+----------+----------+------------+
7 rows in set (0.00 sec)

```

## 9.4 type类型

这里我们主要关注两个指标　

- `ALL`:显示所性能信息
- `BLOCK IO`:显示块(页)IO的数量
  - BLOCK_OPS_IN:输入页数量
  - BLOCK_OPS_OUT:输出页数量
- `CONTEXT SWITCHES`:上下文切换相关开销
- `CPU`:显示CPU相关开销信息
  - CPU_user:用户所用时间,秒单位 
  - CPU_system:系统所用时间,秒单位 
- `IPC`:显示发送和接收相关开销信息
- `MEMORY`:显示内存相关开销信息
- `PAGE FAULTS` :显示页面错误相关开销信息
- `SOURCE` :显示和Source_function，Source_file，Source_line相关的开销信息
- `SWAPS` :显示交换次数相关开销的信息

## 9.5 profile过程

```mysql
mysql> show profile for query 4;
+----------------------+-----------+
| Status               | Duration  |
+----------------------+-----------+
| starting             |  0.000083 |
| checking permissions |  0.000013 |
| Opening tables       |  0.000019 |
| init                 |  0.000036 |
| System lock          |  0.000014 |
| optimizing           |  0.000012 |
| statistics           |  0.000021 |
| preparing            |  0.000018 |
| executing            |  0.000006 |
| Sending data         | 87.327560 |
| end                  |  0.000016 |
| query end            |  0.000013 |
| closing tables       |  0.000011 |
| freeing items        |  0.000027 |
| logging slow query   |  0.000057 |
| cleaning up          |  0.000017 |
+----------------------+-----------+
16 rows in set, 1 warning (0.00 sec)

starting：开始
checking permissions：检查权限
Opening tables：打开表
init ： 初始化
System lock ：系统锁
optimizing ： 优化
statistics ： 统计
preparing ：准备
executing ：执行
Sending data ：发送数据
Sorting result ：排序
end ：结束
query end ：查询 结束
closing tables ： 关闭表 ／去除TMP 表
freeing items ： 释放事件
cleaning up ：清理

profile只能列出使用到的环节　没有使用的环节不显示
```



### 9.5.1 重点环节

- `preparing` ：准备
- `executing` ：执行
- `Sending data` ：发送数据  一般这个环节时间最长
- `Sorting result` ：排序

### 9.5.2 不应/减少出现的环节

- `converting HEAP to MyISAM`： 查询结果太大，内存都不够用了，往磁盘上搬了
- `creating tmp table` ：创建临时表，拷贝数据到临时表，然后再删除
- `copying to tmp table on disk` ：把内存中临时表复制到磁盘
- `locked`: 被锁喉了~

## 9.6 日常常用指令

```mysql
SHOW PROFILE block io,cpu FOR QUERY N;
多注意CPU和IO


mysql> SHOW PROFILE block io,cpu FOR QUERY 4\G;
*************************** 1. row ***************************
       Status: starting
     Duration: 0.000083
     CPU_user: 0.000049
   CPU_system: 0.000025
 Block_ops_in: 0
Block_ops_out: 0
*************************** 2. row ***************************
       Status: checking permissions
     Duration: 0.000013
     CPU_user: 0.000006
   CPU_system: 0.000003
 Block_ops_in: 0
Block_ops_out: 0
*************************** 3. row ***************************
       Status: Opening tables
     Duration: 0.000019
     CPU_user: 0.000012
   CPU_system: 0.000006
 Block_ops_in: 0
Block_ops_out: 0
*************************** 4. row ***************************
       Status: init
     Duration: 0.000036
     CPU_user: 0.000022
   CPU_system: 0.000012
 Block_ops_in: 0
Block_ops_out: 0
*************************** 5. row ***************************
       Status: System lock
     Duration: 0.000014
     CPU_user: 0.000008
   CPU_system: 0.000004
 Block_ops_in: 0
Block_ops_out: 0
*************************** 6. row ***************************
       Status: optimizing
     Duration: 0.000012
     CPU_user: 0.000007
   CPU_system: 0.000003
 Block_ops_in: 0
Block_ops_out: 0
*************************** 7. row ***************************
       Status: statistics
     Duration: 0.000021
     CPU_user: 0.000013
   CPU_system: 0.000007
 Block_ops_in: 0
Block_ops_out: 0
*************************** 8. row ***************************
       Status: preparing
     Duration: 0.000018
     CPU_user: 0.000011
   CPU_system: 0.000006
 Block_ops_in: 0
Block_ops_out: 0
*************************** 9. row ***************************
       Status: executing
     Duration: 0.000006
     CPU_user: 0.000003
   CPU_system: 0.000002
 Block_ops_in: 0
Block_ops_out: 0
*************************** 10. row ***************************
       Status: Sending data
     Duration: 87.327560
     CPU_user: 37.617680
   CPU_system: 32.943640
 Block_ops_in: 2591264
Block_ops_out: 0
*************************** 11. row ***************************
       Status: end
     Duration: 0.000016
     CPU_user: 0.000006
   CPU_system: 0.000004
 Block_ops_in: 0
Block_ops_out: 0
*************************** 12. row ***************************
       Status: query end
     Duration: 0.000013
     CPU_user: 0.000007
   CPU_system: 0.000005
 Block_ops_in: 0
Block_ops_out: 0
*************************** 13. row ***************************
       Status: closing tables
     Duration: 0.000011
     CPU_user: 0.000006
   CPU_system: 0.000004
 Block_ops_in: 0
Block_ops_out: 0
*************************** 14. row ***************************
       Status: freeing items
     Duration: 0.000027
     CPU_user: 0.000015
   CPU_system: 0.000010
 Block_ops_in: 0
Block_ops_out: 0
*************************** 15. row ***************************
       Status: logging slow query
     Duration: 0.000057
     CPU_user: 0.000032
   CPU_system: 0.000021
 Block_ops_in: 0
Block_ops_out: 8
*************************** 16. row ***************************
       Status: cleaning up
     Duration: 0.000017
     CPU_user: 0.000009
   CPU_system: 0.000006
 Block_ops_in: 0
Block_ops_out: 0
16 rows in set, 1 warning (0.00 sec)

ERROR: 
No query specified


```

## 9.7  使用Performance查看profi

通过help profile我们知道show proflie相关指令即将废弃,可以使用如下指令进行查询

在使用以下两个命令前需要做一些设置角色的操作:具体请参考官方文档:

[25.19.1 Query Profiling Using Performance Schema](https://dev.mysql.com/doc/refman/5.7/en/performance-schema-query-profiling.html)



### 9.7.1 查看所有语句

```mysql
mysql> SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT        FROM performance_schema.events_statements_history_long\G;
*************************** 1. row ***************************
EVENT_ID: 146
Duration: 0.000842
SQL_TEXT: UPDATE performance_schema.setup_consumers
       SET ENABLED = 'YES'
       WHERE NAME LIKE '%events_statements_%'
*************************** 2. row ***************************
EVENT_ID: 147
Duration: 0.000376
SQL_TEXT: UPDATE performance_schema.setup_consumers
       SET ENABLED = 'YES'
       WHERE NAME LIKE '%events_stages_%'
*************************** 3. row ***************************
EVENT_ID: 154
Duration: 2.089741
SQL_TEXT: select count(*) from t_scrm_pet_info where pet_birthday > '2019-01-01 00:00:01' and pet_birthday < '2019-03-01 00:00:01'
*************************** 4. row ***************************
EVENT_ID: 171
Duration: 0.003488
SQL_TEXT: SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT
       FROM performance_schema.events_statements_history_long
*************************** 5. row ***************************
EVENT_ID: 188
Duration: 0.000899
SQL_TEXT: SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration
       FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=154
*************************** 6. row ***************************
EVENT_ID: 205
Duration: 0.000480
SQL_TEXT: SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration        FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=154
*************************** 7. row ***************************
EVENT_ID: 222
Duration: 0.000669
SQL_TEXT: SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT        FROM performance_schema.events_statements_history_long
7 rows in set (0.00 sec)

ERROR: 
No query specified

```



### 9.7.2 查看指定语句

``` mysql
mysql> SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration        FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=154\G;
*************************** 1. row ***************************
   Stage: stage/sql/starting
Duration: 0.000105
*************************** 2. row ***************************
   Stage: stage/sql/checking permissions
Duration: 0.000008
*************************** 3. row ***************************
   Stage: stage/sql/Opening tables
Duration: 0.000021
*************************** 4. row ***************************
   Stage: stage/sql/init
Duration: 0.000059
*************************** 5. row ***************************
   Stage: stage/sql/System lock
Duration: 0.000016
*************************** 6. row ***************************
   Stage: stage/sql/optimizing
Duration: 0.000014
*************************** 7. row ***************************
   Stage: stage/sql/statistics
Duration: 0.000023
*************************** 8. row ***************************
   Stage: stage/sql/preparing
Duration: 0.000017
*************************** 9. row ***************************
   Stage: stage/sql/executing
Duration: 0.000002
*************************** 10. row ***************************
   Stage: stage/sql/Sending data
Duration: 2.089357
*************************** 11. row ***************************
   Stage: stage/sql/end
Duration: 0.000005
*************************** 12. row ***************************
   Stage: stage/sql/query end
Duration: 0.000012
*************************** 13. row ***************************
   Stage: stage/sql/closing tables
Duration: 0.000009
*************************** 14. row ***************************
   Stage: stage/sql/freeing items
Duration: 0.000027
*************************** 15. row ***************************
   Stage: stage/sql/logging slow query
Duration: 0.000055
*************************** 16. row ***************************
   Stage: stage/sql/cleaning up
Duration: 0.000001
16 rows in set (0.00 sec)

ERROR: 
No query specified

```
