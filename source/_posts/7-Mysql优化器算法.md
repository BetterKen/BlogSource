---
title: 7.Mysql优化器算法
date: 2020-02-16 19:17:10
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---
除了我们自己手动进行优化外,根据之前的章节,我们可以知道,Mysql内部自己有优化器,对SQL语句进行一些内部的优化,本章介绍几个常用的优化算法:

- MRR
- ICP
- BKA

## 7.1 MRR

MRR——Multi Range Read

### 7.1.1作用

减少磁盘的随机访问，并将随机访问转化为顺序的数据访问

### 7.1.2 原理

在不使用 MRR 时，优化器需要根据二级索引返回的记录来进行“回表”，这个过程一般会有较多的随机 IO, 使用 MRR 时，SQL 语句的执行过程是这样的：

1. 优化器将二级索引查询到的记录放到一块缓冲区中；
2. 如果二级索引扫描到文件的末尾或者缓冲区已满，则使用`快速排序对缓冲区中的内容按照主键进行排序`；
3. 用户线程调用 MRR 接口取 cluster index，然后根据cluster index 取行数据；
4. 当根据缓冲区中的 cluster index 取完数据，则继续调用过程 2) 3)，直至扫描结束；

通过上述过程，优化器将二级索引随机的 IO 进行排序，转化为主键的有序排列，从而实现了随机 IO 到顺序 IO 的转化，提升性能



在未开启MRR时,查询方式如图所示:

![](http://mysql317.oss-cn-beijing.aliyuncs.com/no-mrr-access-pattern.png)



开启MRR后,查询方式如下图所示:

![](http://mysql317.oss-cn-beijing.aliyuncs.com/mrr-access-pattern.png)



### 7.1.3 开关MRR

我们可以通过 以下命令查看和开启MRR,5.6以后默认是开启的

```mysql
查看MRR是否开启
mysql> select @@optimizer_switch\G
*************************** 1. row ***************************
@@optimizer_switch: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on
1 row in set (0.00 sec)

注意上面的mrr=on

关闭MRR
mysql> set optimizer_switch='mrr=off';
开启MRR
mysql> set optimizer_switch='mrr=on';

相关参数
当mrr=on,mrr_cost_based=on，则表示cost base的方式还选择启用MRR优化,当发现优化后的代价过高时就会不使用该项优化
当mrr=on,mrr_cost_based=off，则表示总是开启MRR优化

尽量设置 mrr_cost_based=ON，毕竟大多数情况下优化器是对的

```

### 7.1.4 开关对比

```mysql
mysql> explain select *  from t_scrm_user_info where bigdata_user_id > 'A1001036' and bigdata_user_id < 'A2100000';
```

当`mrr=off`时

![](http://mysql317.oss-cn-beijing.aliyuncs.com/mrroffexplain.png)

当`mrr=on`时

![](http://mysql317.oss-cn-beijing.aliyuncs.com/mrrontime.png)

可以看到当mrr开启时, extra 的输出中多了 “Using MRR” 信息，即使用了 MRR Optimization IO 层面进行了优化，减少 IO 方面的开销

在时间上理论上可以查一个数量级,但是要在缓存池中没有预热以及查询的数据不在缓冲池中才可以

我测试的时候时间都差不多 (⊙﹏⊙)b



## 7.2 ICP

ICP——Index Condition Pushdown

### 7.2.1作用

在mysql数据库取出索引的同时,判断是否可以进行WHERE条件的过滤,也就是讲WHERE条件的过滤放到了存储引擎层,大大减少了上层SQL层对记录的fetch索取,以提高性能

### 7.2.2 原理

5.6 之前，在 SQL 语句的执行过程中，server 层通过 engine 的 api 获取数据，然后再进行 where_cond 的判断（具体判断逻辑在: evaluate_join_record），每一条数据都需要从engine层返回server层做判断。我们回顾一下上面把 ICP 关掉的测试，可以看到 Handler_read_next 的值陡增，其原因是第 1 个字段区分度不高，且 memo 字段无法使用索引，造成了类似 index 扫描的的情况，性能较低。

5.6 之后，在利用索引扫描的过程中，如果发现 where_cond 中含有这个 index 相关的条件，则将此条件记录在 handler 接口中，在索引扫描的过程中，只有满足索引与handler接口的条件时，才会返回到 server 层做进一步的处理，在前缀索引区分度不够，其它字段区分度高的情况下可以有效的减少 server & engine之间的开销，提升查询性能。

开启ICP前查询方式如图所示:

![](http://mysql317.oss-cn-beijing.aliyuncs.com/index-access-2phases.png)



开启ICP后查询方式如图所示:

![](http://mysql317.oss-cn-beijing.aliyuncs.com/index-access-with-icp.png)

### 7.2.3 开关ICP

```mysql
默认是开启的
查看MRR是否开启
mysql> select @@optimizer_switch\G
*************************** 1. row ***************************
@@optimizer_switch: index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on
1 row in set (0.00 sec)

注意上面的index_condition_pushdown=on

SET optimizer_switch='index_condition_pushdown=on'
SET optimizer_switch='index_condition_pushdown=off'

```



### 7.2.4 开关对比

开启ICP时,使用explain语句时extra字段有`Using index condition`关键字

![](http://mysql317.oss-cn-beijing.aliyuncs.com/mrroffexplain.png)



### 7.2.5 使用场景

- 只支持 select 语句；

- MyISAM 与 InnoDB 引擎都起作用;

- ICP的优化策略可用于range、ref、eq_ref、ref_or_null 类型的访问数据方法；

- 不支持主建索引，只支持辅助索引；

- 涉及子查询的不能起作用

- 作用于多列索引的情况最为明显

  ```mysql
  SELECT * FROM people
    WHERE zipcode='95054'
    AND lastname LIKE '%etrunia%'
    AND address LIKE '%Main Street%';
  ```



## 7.3 BKA

BKA——Batched Key Access

BKA的前辈是BNL:

### 7.3.1 Block Nested-Loop Join算法

将外层循环的行/结果集存入join buffer, 内层循环的每一行与整个buffer中的记录做比较，从而减少内层循环的次数。主要用于当被join的表上无索引。

### 7.3.2 Batched Key Access算法

当被join的表能够使用索引时，就先好顺序，然后再去检索被join的表。对这些行按照索引字段进行排序，因此减少了随机IO。如果被Join的表上没有索引，则使用老版本的BNL策略(BLOCK Nested-loop)。



### 7.3.3 BKA和BNL标识

Explain下的Extra显示不同：

- Using join buffer (Batched Key Access)

- Using join buffer (Block Nested Loop)

  

```mysql
相关参数

BAK使用了MRR，要想使用BAK必须打开MRR功能，而MRR基于mrr_cost_based的成本估算并不能保证总是使用MRR，官方推荐设置mrr_cost_based=off来总是开启MRR功能。打开BAK功能(BAK默认OFF)：

SET optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';

BKA使用join buffer size来确定buffer的大小，buffer越大，访问被join的表/内部表就越顺序。

BNL默认是开启的，设置BNL相关参数：

SET optimizer_switch=’block_nested_loop’

支持inner join, outer join, semi-join operations,including nested outer joins

```

`BKA主要适用于join的表上有索引可利用，无索引只能使用BNL`



### 7.3.4 BKA BNL MRR的关系

BKA = BNL + MRR

![](http://mysql317.oss-cn-beijing.aliyuncs.com/bka.jpg)
