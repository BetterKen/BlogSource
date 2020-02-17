---
title: 8.MysqlExplain介绍
date: 2020-02-16 19:18:10
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---

## 8.1 Explain各列意义

EXPLAIN 命令的输出内容大致如下:

```mysql
mysql> explain select *  from t_scrm_user_info where bigdata_user_id > 'A1001036' and bigdata_user_id < 'A2100000'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_user_info
   partitions: NULL
         type: range
possible_keys: UQ_CUSTOMER_ID
          key: UQ_CUSTOMER_ID
      key_len: 130
          ref: NULL
         rows: 168902
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)
```



|列名称|含义|
|:----:|:----:|
|id| SELECT 查询的标识符. 每个 SELECT 都会自动分配一个唯一的标识符.|
|select_type| SELECT 查询的类型.|
|table| 查询的是哪个表|
|partitions| 匹配的分区|
|type| join 类型|
|possible_keys| 此次查询中可能选用的索引|
|key| 此次查询中确切使用到的索引.|
|ref| 哪个字段或常数与 key 一起被使用|
|rows| 显示此查询一共扫描了多少行. 这个是一个估计值.|
|filtered| 表示此查询条件所过滤的数据的百分比|
|extra| 额外的信息|

## 8.2 id

表示select标识符，同时表明执行顺序，也就是说id是一个查询的序列号，查询序号即为sql语句执行的顺序。

1. 当id值相同时，按从上到下的顺序执行
2. 当id全部不同时，按id从大到小执行
3. 当id部分不同时，先执行id大的，id相同的，按从上到下的顺序执行



## 8.3 select_type

1. SIMPLE 简单的select查询，查询中不包含子查询或者UNION
2. PRIMARY 查询中若包含任何复杂的子部分，最外层查询则被标记为PRIMARY
3. SUBQUERY 在SELECT或WHERE列表中包含了子查询
4. DERIVED 在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL会递归执行这些子查询，把结果放在临时表中
5. UNION 若第二个SELECT出现在UNION之后，则被标记为UNION：若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED
6. UNION RESULT 从UNION表获取结果的SELECT

```mysql
mysql> explain select * from t_scrm_user_info where  id = 1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_user_info
   partitions: NULL
         type: const
possible_keys: PRIMARY,IX_ID_CREATE_TIME
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)

ERROR: 
No query specified
==================================================================================
mysql> explain select * from (select * from t_scrm_pet_info limit 20) t where  t.id = (select id from t_scrm_map limit 1)\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
   partitions: NULL
         type: ref
possible_keys: <auto_key0>
          key: <auto_key0>
      key_len: 4
          ref: const
         rows: 2
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 3
  select_type: SUBQUERY
        table: t_scrm_map
   partitions: NULL
         type: index
possible_keys: NULL
          key: IX_CREATE_TIME
      key_len: 5
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: Using index
*************************** 3. row ***************************
           id: 2
  select_type: DERIVED
        table: t_scrm_pet_info
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 4578062
     filtered: 100.00
        Extra: NULL
3 rows in set, 1 warning (0.00 sec)

ERROR: 
No query specified

==================================================================================

mysql> explain select * from t_scrm_map where id =10 union select * from t_scrm_map where id = 20\G;
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: t_scrm_map
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 2
  select_type: UNION
        table: t_scrm_map
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 3. row ***************************
           id: NULL
  select_type: UNION RESULT
        table: <union1,2>
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Using temporary
3 rows in set, 1 warning (0.00 sec)

```





## 8.4 table

这一列表示 explain 的一行正在访问哪个表。

当 from 子句中有子查询时，table列是 <derivenN> 格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。当有 union 时，UNION RESULT 的 table 列的值为 <union1,2>，1和2表示参与 union 的 select 行id。

## 8.5 partitions 

使用的哪些分区（对于非分区表值为null）

## 8.6 type

type所显示的是查询使用了哪种类型，type包含的类型包括如下图所示的几种：

从最好到最差依次是：

```
system > const > eq_ref > ref > range > index > all
```



一般来说，得保证查询至少达到range级别，最好能达到ref。

- `const, system`：mysql能对查询的某部分进行优化并将其转化成一个常量。用于 primary key 或 unique key 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。

```mysql
mysql> explain select * from t_scrm_user_info where bigdata_user_id = 'B13470427'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_user_info
   partitions: NULL
         type: const
possible_keys: UQ_CUSTOMER_ID
          key: UQ_CUSTOMER_ID
      key_len: 130
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)

```

- `eq_ref`：primary key 或 unique key 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。这可能是在 const 之外最好的`联接类型`了，简单的 select 查询不会出现这种 type。

```mysql
mysql> explain select * from t_scrm_user_info u left join t_scrm_map m on u.bigdata_user_id = m.bigdata_id where u.bigdata_user_id = 'B13470427'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: u
   partitions: NULL
         type: const
possible_keys: UQ_CUSTOMER_ID
          key: UQ_CUSTOMER_ID
      key_len: 130
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: m
   partitions: NULL
         type: ref
possible_keys: UQ_ID
          key: UQ_ID
      key_len: 258
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using where
2 rows in set, 1 warning (0.00 sec)

```

- ref：相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行。

```mysql
mysql> explain select * from t_scrm_map where target_id = '11255964'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ref
possible_keys: IX_TRAGET_ID
          key: IX_TRAGET_ID
      key_len: 130
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

- `ref_or_null`：类似ref，但是可以搜索值为NULL的行。
- `range`：范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。

```mysql
mysql> explain select * from t_scrm_map where id > 10 and id  <20\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 9
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

ERROR: 
No query specified

```

- `index`：和ALL一样，不同就是mysql只需扫描索引树，这通常比ALL快一些。

```mysql
mysql> explain select count(*) from t_scrm_map\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: index
possible_keys: NULL
          key: IX_CREATE_TIME
      key_len: 5
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

ERROR: 
No query specified

```

- `ALL`：即全表扫描，意味着mysql需要从头到尾去查找所需要的行。通常情况下这需要增加索引来进行优化了

```mysql
mysql> explain select * from t_scrm_map\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)

ERROR: 
No query specified

```



## 8.7  possible_keys

这一列显示查询可能使用哪些索引来查找。 

explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引对此查询帮助不大，选择了全表查询。 

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提高查询性能，然后用 explain 查看效果。

## 8.8 key

这一列显示mysql实际采用哪个索引来优化对该表的访问。

如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索引，在查询中使用 force index、ignore index。

## 8.9 key_len列

这一列显示了mysql在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列。 

```mysql
key_len 的计算规则如下:

字符串
char(n): n 字节长度
varchar(n): 如果是 utf8 编码, 则是 3 n + 2字节; 如果是 utf8mb4 编码, 则是 4n +2 字节.

数值类型:
TINYINT: 1字节
SMALLINT: 2字节
MEDIUMINT: 3字节
INT: 4字节
BIGINT: 8字节

时间类型
DATE: 3字节
TIMESTAMP: 4字节
DATETIME: 8字节

字段属性: 
NULL 属性 占用一个字节. 如果一个字段是 NOT NULL 的, 则没有此属性.
```

```mysql
mysql> explain select count(*) from t_scrm_map where target_id  = '11255964'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ref
possible_keys: IX_TRAGET_ID
          key: IX_TRAGET_ID
      key_len: 130
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

/**
| t_scrm_map | CREATE TABLE `t_scrm_map` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `bigdata_id` varchar(64) NOT NULL COMMENT '唯一ID',
  `type` tinyint(3) NOT NULL COMMENT '关联类型',
  PRIMARY KEY (`id`),
  UNIQUE KEY `UQ_ID` (`bigdata_id`,`type`),
  KEY `IX_MAP_ID` (`scrm_id`),
) ENGINE=InnoDB AUTO_INCREMENT=11962156 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='关系表' |
*/


mysql> explain select count(*) from t_scrm_map where bigdata_id  = '11255964'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ref
possible_keys: UQ_ID
          key: UQ_ID
      key_len: 258
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.04 sec)

可以看到key_len = 258---> 64*4+2;即只使用了索引UQ_ID的bigdata_id部分


mysql> explain select * from t_scrm_map where bigdata_id  = 'B11255964' and type = 1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: const
possible_keys: UQ_ID
          key: UQ_ID
      key_len: 259
          ref: const,const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.04 sec)

可以看到key_len = 258---> 64*4+2+1;即使用了索引UQ_ID的bigdata_id和type部分

```

## 8.10 ref列

这一列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有：

- const（常量）

- func

- NULL

- 字段名

  

## 8.11 rows列rows

rows 也是一个重要的字段. MySQL 查询优化器根据统计信息, 估算 SQL 要查找到结果集需要扫描读取的数据行数. 这个值非常直观显示 SQL 的效率好坏, 原则上 rows 越少越好.



## 8.12 Extra列

这一列展示的是额外信息。常见的重要值如下： 

### 8.12.1 Using filesort	

mysql 会对结果使用一个外部索引排序，而不是按索引次序从表里读取行。此时mysql会根据联接类型浏览所有符合条件的记录，并保存排序关键字和行指针，然后排序关键字并按顺序检索行信息。

`这种情况是要考虑使用索引来优化的`

```mysql
mysql> explain select * from t_scrm_map order by type asc\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
```



### 8.12.2 Using temporary

使用临时表保存中间结果，常用于GROUP BY 和 ORDER BY,DISTINCT操作中。

出现这种情况一般是要进行优化的，首先是想到用索引来优化。

```mysql
mysql> explain select distinct(type) from t_scrm_map order by type asc\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: index
possible_keys: UQ_ID
          key: UQ_ID
      key_len: 259
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: Using index; Using temporary; Using filesort
1 row in set, 1 warning (0.00 sec)

```

### 8.12.3 Using where	

在查找使用索引的情况下，需要回表去查询所需的数据

```mysql
mysql> explain select * from t_scrm_map where target_id  = '11255964' and type =2\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: ref
possible_keys: IX_TRAGET_ID
          key: IX_TRAGET_ID
      key_len: 130
          ref: const
         rows: 1
     filtered: 10.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)

```

### 8.12.4 Using index	

这发生在对表的请求列都是同一索引的部分的时候，返回的列数据只使用了索引中的信息，而没有再去访问表中的行记录。是性能高的表现。

```mysql
mysql> explain select count(*) from t_scrm_map\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: index
possible_keys: NULL
          key: IX_CREATE_TIME
      key_len: 5
          ref: NULL
         rows: 11271619
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

```

### 8.12.5 Using index condition	

这是MySQL 5.6出来的新特性，叫做“索引条件推送”。

简单说一点就是MySQL原来在索引上是不能执行如like这样的操作的，但是现在可以了，这样减少了不必要的IO操作，但是只能用在二级索引上。

```mysql
mysql> explain select * from t_scrm_map where bigdata_id  = 'B11255964' or bigdata_id like 'B125294%'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_map
   partitions: NULL
         type: range
possible_keys: UQ_ID
          key: UQ_ID
      key_len: 258
          ref: NULL
         rows: 8
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.00 sec)


```



###  8.12.6 Using join buffer	

使用了连接缓存：

- Block Nested Loop，连接算法是块嵌套循环连接;
- Batched Key Access，连接算法是批量索引连接



### 8.12.7 Using MRR

使用了MRR优化算法

```mysql
mysql> explain select *  from t_scrm_user_info where bigdata_user_id > 'A1001036' and bigdata_user_id < 'A2100000'\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t_scrm_user_info
   partitions: NULL
         type: range
possible_keys: UQ_CUSTOMER_ID
          key: UQ_CUSTOMER_ID
      key_len: 130
          ref: NULL
         rows: 168902
     filtered: 100.00
        Extra: Using index condition; Using MRR
1 row in set, 1 warning (0.00 sec)

```



