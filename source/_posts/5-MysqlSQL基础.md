---
title: 5.MysqlSQL基础
date: 2020-02-15 13:15:18
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---

## 5.1 常用数据类型

### 5.1.1 整型

|  类型   | 字节  |      范围(有符号位)       |
| :-----: | :---: | :-----------------------: |
| TINYINT | 1Byte |  0~2^8 OR  -2^7 ~ 2^7-1   |
|   INT   | 4Byte | 0~2^32 OR  -2^31 ~ 2^31-1 |
| BIGINT  | 8Byte | 0~2^64 OR -2^63 ~ 2^63-1  |

- int(M)  M表示总位数
  - 如果某个数不够定义字段时设置的位数，则前面以0补填，zerofill 属性修改
  - 例：int(5)   插入一个数'123'，补填后为'00123'
- 默认存在符号位，unsigned 属性修改
- 在满足要求的情况下，越小越好
- 1表示bool值真，0表示bool值假。MySQL没有布尔类型，通过整型0和1表示。常用tinyint(1)表示布尔型

### 5.1.2 小数

|      类型      |  字节  |   范围(有符号位)    |
| :------------: | :----: | :-----------------: |
| float(单精度)  | 4Byte  | 自定义，表示近似值  |
| double(双精度) | 8Byte  | 自定义，表示近似值  |
|    decimal     | 自定义 | 自定义,表示精确数值 |

- 浮点型既支持符号位 unsigned 属性，也支持显示宽度 zerofill 属性，不同于整型，前后均会补填0
- 支持科学计数法表示。

- 定义浮点型时，需指定总位数和小数位数。

  > float(M, D)     double(M, D)
  >
  > M表示总位数，D表示小数位数
  >
  > M和D的大小会决定浮点数的范围 不同于整型的固定范围
  >
  > M既表示总位数（不包括小数点和正负号），也表示显示宽度（所有显示符号均包括）
  >
  > decimal(M, D)   M也表示总位数，D表示小数位数
  >
  > 保存一个精确的数值，不会发生数据的改变，不同于浮点数的四舍五入
  > 将浮点数转换为字符串来保存，每9位数字保存为4个字节

### 5.1.3 字符

|  类型   | 说明                                                         |
| :-----: | :----------------------------------------------------------- |
|  CHAR   | 定长字符串最多255个字符                                      |
| VARCHAR | 变长字符串最多65535个字符                                    |
|  TEXT   | 变长字符串最多65535个字符类型，在定义时,不需要定义长度<br/>也不会计算总长度不可给default值 |
|  BLOB   | 二进制字符串（字节字符串）                                   |
|  ENUM   | 枚举类型 smallint 存储                                       |
|   SET   | 集合类型 bigint存储                                          |

```
char(11) ：
定长字符串类型,在存储字符串时，最大字符长度11个，立即分配11个字符长度的存储空间，如果存不满，空格填充
varchar(11):
变长的字符串类型看，最大字符长度11个。在存储字符串时，自动判断字符长度，按需分配存储空间。
M表示能存储的最大长度，此长度是字符数，非字节数。
不同的编码，所占用的空间不同。
char,最多255个字符，与编码无关。
varchar,最多65535字符，与编码有关。
一条有效记录最大不能超过65535个字节。
utf8 最大为21844个字符，gbk 最大为32766个字符，latin1 最大为65532个字符
【注】varchar 是变长的，需要利用存储空间保存 varchar 的长度，如果数据小于255个字节，则采用一个字节来保存长度，反之需要两个字节来保存。
varchar 的最大有效长度由最大行大小和使用的字符集确定。

最大有效长度是65532字节，因为在varchar存字符串时，第一个字节是空的，不存在任何数据，然后还需两个字节来存放字符串的长度，所以有效长度是64432-1-2=65532字节。

例：若一个表定义为 CREATE TABLE tb(c1 int, c2 char(30), c3 varchar(N)) charset=utf8; 问N的最大值是多少？
答：(65535-1-2-4-30*3)/3

```

```
枚举类型说明
enum(val1, val2, val3...)
在已知的值中进行单选。最大数量为65535.
枚举值在保存时，以2个字节的整型(smallint)保存。每个枚举值，按保存的位置顺序，从1开始逐一递增。
表现为字符串类型，存储却是整型。
NULL值的索引是NULL。
空字符串错误值的索引值是0
枚举类型，比较适合于将来此列的值是固定范围内的特点，可以使用enum,可以很大程度的优化我们的索引结构
```

   ```mysql
集合类型说明
set(val1, val2, val3...)
create table tab ( gender set('男', '女', '无') );
insert into tab values ('男, 女');
最多可以有64个不同的成员。以bigint存储，共8个字节。采取位运算的形式
当创建表时，SET成员值的尾部空格将自动被删除
   ```



### 5.1.4 时间

|   类型    | 字节  |    说明    |                    范围                    |
| :-------: | :---: | :--------: | :----------------------------------------: |
| datetime  | 8Byte | 日期及时间 | 1000-01-01 00:00:00 到 9999-12-31 23:59:59 |
|   date    | 3Byte |    日期    |          1000-01-01 到 9999-12-31          |
| timestamp | 4Byte |   时间戳   |   19700101000000 到 2038-01-19 03:14:07    |
|   time    | 3Byte |    时间    |          -838:59:59 到 838:59:59           |
|   year    | 1Byte |    年份    |                1901 - 2155                 |



## 5.2 列属性

###  5.2.1 PRIMARY

- 能唯一标识记录的字段，可以作为主键。
- 一个表只能有一个主键。
- 主键具有唯一性。
- 声明字段时，用 primary key 标识。
- 也可以在字段列表之后声明：create table tab ( id int, stu varchar(10), primary key (id));
- 主键字段的值不能为null。
- 主键可以由多个字段共同组成。此时需要在字段列表后声明的方法。
    例：create table tab ( id int, stu varchar(10), age int, primary key (stu, age));

### 5.2.2 UNIQUE
唯一索引,使得某字段的值也不能重复。

### 5.2.3 NULL
- null不是数据类型，是列的一个属性。
- 表示当前列是否可以为null，表示什么都没有。
- null, 允许为空。默认；not null, 不允许为空。
- insert into tab values (null, 'val'); -- 此时表示将第一个字段的值设为null, 取决于该字段是否允许为null

### 5.2.4 DEFAULT
- 当前字段的默认值。
```mysql
#表示强制使用默认值
insert into tab values (default, 'val');
#表示将当前时间的时间戳 设为默认值
create table tab ( add_time timestamp default current_timestamp )
```

### 5.2.5 AUTO_INCREMENT
- 自动增长约束
- 自动增长必须为索引（主键或unique）
- 只能存在一个字段为自动增长。
- 默认为1开始自动增长。可以通过表属性 auto_increment = x进行设置，或 alter table tbl auto_increment = x;

### 5.2.6 COMMENT
注释
例：create table tab ( id int ) comment '注释内容';
### 5.2.7 FOREIGN KEY 
外键约束 高并发下不建议使用 ERP中常用



## 5.3 表属性

### 5.3.1 engine

使用的存储引擎 建议都是用Innodb

### 5.3.2 charset

使用的字符集 这个参数也可以在列属性里设置,但是不建议这样做,整表设置相同字符集即可

常用的有:

- utf8 推荐使用** 
- utf8mb4 推荐使用***
- gbk

建议默认使用utf8mb4格式:

>MySQL在5.5.3之后增加了utf8mb4的编码，mb4即4-Byte UTF-8 Unicode Encoding，专门用来兼容四字节的unicode。utf8mb4为utf8的超集并兼容utf8，比utf8能表示更多的字符。
>
>低版本的MySQL支持的utf8编码，最大字符长度为 3 字节，如果遇到 4 字节的字符就会出现错误了。
>
>常见的四字节就是Emoji 表情（Emoji 是一种特殊的 Unicode 编码，常见于 ios 和 android 手机上），和一些不常用的汉字，以及任何新增的 Unicode 字符等等。



## 5.4 常用SQL分类

- DDL：数据定义语言
- DML：数据操作语言
- DCL：数据控制语言
- DQL：数据的查询语言



## 5.5 DDL语句

### 5.5.1 数据库操作

```mysql
# 创建demo数据库
mysql> create database demo charset utf8mb4
mysql> create database if not exists demo charset utf8mb4;

# 删除demo数据库
mysql> drop database demo;
mysql> drop database if exists demo;

# 修改database字符集
mysql> alter database demo charset=utf8;
```

### 5.5.2 表操作

```mysql
# 创建表
CREATE TABLE `t_scrm_user_info`
(
  `id`          INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `user_id`     VARCHAR(32)      NOT NULL COMMENT '用户ID作为唯一标示',
  `user_name`   VARCHAR(32)      NOT NULL DEFAULT '' COMMENT '用户姓名',
  `user_sex`    TINYINT(3)       NOT NULL DEFAULT 0 COMMENT '用户性别 0未知 1男 2女',
  `user_mobile` VARCHAR(16)      NOT NULL DEFAULT '' COMMENT '用户手机号',
  `user_status` TINYINT(3)       NOT NULL DEFAULT 0 COMMENT '用户状态 0正常 1锁定 -1删除',
  `user_avatar` varchar(256)     NOT NULL DEFAULT '' COMMENT '用户头像',
  `create_time` DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `UQ_USER_ID` (`user_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4 COMMENT ='用户信息表';
 
# 删除表
mysql> drop table t_scrm_user_info;

# 重命名表
mysql> RENAME TABLE t_scrm_user_info to demo;

# 复制表
mysql> CREATE table t_scrm_user_info like demo;

# 清空表
mysql> truncate table t_scrm_user_info;

# 添加列——添加update_time列
ALTER TABLE t_scrm_user_info ADD update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间';

# 添加列——在user_avatar列后添加user_remark列
ALTER TABLE t_scrm_user_info ADD `user_remark` VARCHAR(128) NOT NULL DEFAULT '' COMMENT '用户备注' AFTER user_avatar;

# 更新列
mysql> ALTER TABLE t_scrm_user_info MODIFY user_name VARCHAR(64);
注意上面语句更新的时候 会把user_name列上的其他属性更新没 如not null属性和comment属性
所以建议写成如下：
mysql> ALTER TABLE t_scrm_user_info MODIFY user_name VARCHAR(32)      NOT NULL DEFAULT '' COMMENT '用户姓名';

# 更新列名
mysql> ALTER TABLE t_scrm_user_info CHANGE user_name u_name VARCHAR(32)      NOT NULL DEFAULT '' COMMENT '用户姓名';

# 删除列
ALTER TABLE t_scrm_user_info DROP user_avatar;

#添加索引
ALTER TABLE tbl_name ADD INDEX index_name (column_list): 添加普通索引，索引值可出现多次。

#删除索引
DROP INDEX [indexName] ON mytable;
```



## 5.6 DML语句

```mysql
# 插入
INSERT INTO t_scrm_user_info
(user_id,user_name,user_sex,user_mobile,user_status) 
VALUES
("123",'nihao',1,'18611111111','0'),
("345",'hahah',1,'18611111111','0');

# 更新
UPDATE t_scrm_user_info set user_id="444" WHERE user_id="123";

# 删除
mysql> DELETE FROM t_scrm_user_info where user_id ="444";

# 全表删除

mysql> DELETE FROM t_scrm_user_info;

与truncate区别
delete: 
		DML操作, 是逻辑性质删除,逐行进行删除,速度慢. 还在文件里 文件大小不会改变
		主键继续原来的增长计数
truncate: 
		DDL操作,对与表段中的数据页进行清空,速度快. 从文件里删除 文件大小改变
		主键从1开始计数
```



## 5.7 DCL语句

```mysql
# 管理用户
通配符：%表示可以在任意主机使用用户登录数据库
【注意】在mysql里面用户的定义为:用户名+主机名
【注意】在mysql8.0中添加用户和授权必须是两步进行 不能用一条语句
# 添加用户
mysql> CREATE USER 'read'@'10.15.2.%' identified by '123';

# 删除用户
mysql> DROP USER 'read'@'10.15.2.%';

#修改用户密码
mysql> set password for 'read'@'10.15.2.%' = password('456');

# 授权
# 查看授权
mysql> show grants for 'read'@'10.15.2.%';
+------------------------------------------+
| Grants for read@10.15.2.%                |
+------------------------------------------+
| GRANT USAGE ON *.* TO 'read'@'10.15.2.%' |
+------------------------------------------+
1 row in set (0.00 sec)

# 添加授权
grant 权限列表 on 数据库.表名  to  '用户名'@'主机名';
mysql> GRANT CREATE,ALTER,DROP,INSERT,UPDATE,DELETE,SELECT ON demo.* TO 'read'@'10.15.2.%';
# 授权所有权限
mysql> grant all privileges on demo.* to 'read'@'10.15.2.%';


# 删除授权
revoke 权限列表  on  数据库.表名  from  '用户名'@'主机名';
mysql> revoke DROP,DELETE  on  demo.* from 'read'@'10.15.2.%';

# 让授权生效
FLUSH PRIVILEGES;

```



## 5.8 DQL语句

### 5.8.1 SELECT

```mysql
SELECT [ALL|DISTINCT] select_expr FROM -> WHERE -> GROUP BY [合计函数] -> HAVING -> ORDER BY -> LIMIT
a. select_expr
    -- 可以用 * 表示所有字段。
        select * from tb;
    -- 可以使用表达式（计算公式、函数调用、字段也是个表达式）
        select stu, 29+25, now() from tb;
    -- 可以为每个列使用别名。适用于简化列标识，避免多个列标识符重复。
        - 使用 as 关键字，也可省略 as.
        select stu+10 as add10 from tb;
b. FROM 子句
    用于标识查询来源。
    -- 可以为表起别名。使用as关键字。
        SELECT * FROM tb1 AS tt, tb2 AS bb;
    -- from子句后，可以同时出现多个表。
        -- 多个表会横向叠加到一起，而数据会形成一个笛卡尔积。
        SELECT * FROM tb1, tb2;
    -- 向优化符提示如何选择索引
        USE INDEX、IGNORE INDEX、FORCE INDEX
        SELECT * FROM table1 USE INDEX (key1,key2) WHERE key1=1 AND key2=2 AND key3=3;
        SELECT * FROM table1 IGNORE INDEX (key3) WHERE key1=1 AND key2=2 AND key3=3;
c. WHERE 子句
    -- 从from获得的数据源中进行筛选。
    -- 整型1表示真，0表示假。
    -- 表达式由运算符和运算数组成。
        -- 运算数：变量（字段）、值、函数返回值
        -- 运算符：
            =, <=>, <>, !=, <=, <, >=, >, !, &&, ||,
            in (not) null, (not) like, (not) in, (not) between and, is (not), and, or, not, xor
            is/is not 加上ture/false/unknown，检验某个值的真假
            <=>与<>功能相同，<=>可用于null比较
d. GROUP BY 子句, 分组子句
    GROUP BY 字段/别名 [排序方式]
    分组后会进行排序。升序：ASC，降序：DESC
    以下[合计函数]需配合 GROUP BY 使用：
    count 返回不同的非NULL值数目  count(*)、count(字段)
    sum 求和
    max 求最大值
    min 求最小值
    avg 求平均值
    group_concat 返回带有来自一个组的连接的非NULL值的字符串结果。组内字符串连接。
e. HAVING 子句，条件子句
    与 where 功能、用法相同，执行时机不同。
    where 在开始时执行检测数据，对原数据进行过滤。
    having 对筛选出的结果再次进行过滤。
    having 字段必须是查询出来的，where 字段必须是数据表存在的。
    where 不可以使用字段的别名，having 可以。因为执行WHERE代码时，可能尚未确定列值。
    where 不可以使用合计函数。一般需用合计函数才会用 having
    SQL标准要求HAVING必须引用GROUP BY子句中的列或用于合计函数中的列。
f. ORDER BY 子句，排序子句
    order by 排序字段/别名 排序方式 [,排序字段/别名 排序方式]...
    升序：ASC，降序：DESC
    支持多个字段的排序。
g. LIMIT 子句，限制结果数量子句
    仅对处理好的结果进行数量限制。将处理好的结果的看作是一个集合，按照记录出现的顺序，索引从0开始。
    limit 起始位置, 获取条数
    省略第一个参数，表示从索引0开始。limit 获取条数
h. DISTINCT, ALL 选项
    distinct 去除重复记录
    默认为 all, 全部记录
```

### 5.8.2 多表查询

多表查询使用join关键字进行连接

连接方式分为四种:

- INNER JOIN：如果表中有至少一个匹配，则返回行
- LEFT JOIN：即使右表中没有匹配，也从左表返回所有的行
- RIGHT JOIN：即使左表中没有匹配，也从右表返回所有的行
- FULL JOIN：只要其中一个表中存在匹配，则返回行

这里不详细介绍 看图理解

![](http://mysql317.oss-cn-beijing.aliyuncs.com/sql-join.png)

也可参考此篇[文档](https://www.runoob.com/sql/sql-join.html)

### 5.8.3 SHOW

```mysql
show  databases;                          #查看所有数据库
show tables;                              #查看当前库的所有表
SHOW TABLES FROM                          #查看某个指定库下的表
show create database world                #查看建库语句
show create table world.city              #查看建表语句
show  grants for  root@'localhost'        #查看用户的权限信息
show  charset；                            #查看字符集
show collation                             #查看校对规则
show processlist;                          #查看数据库连接情况
show index from                            #表的索引情况
show status                                 #数据库状态查看
SHOW STATUS LIKE '%lock%';                 #模糊查询数据库某些状态
SHOW VARIABLES                             #查看所有配置信息
SHOW variables LIKE '%lock%';              #查看部分配置信息
show engines                                #查看支持的所有的存储引擎
show engine innodb status                  #查看InnoDB引擎相关的状态信息
show binary logs                            #列举所有的二进制日志
show master status                           #查看数据库的日志位置信息
show binlog events;                          #查看二进制日志事件
show slave status                           #查看从库状态
SHOW RELAYLOG EVENTS                        #查看从库relaylog事件信息
desc  (show colums from city)               #查看表的列定义信息

```

### 5.8.4其它关键字

```mysql
# distinct：去重复
SELECT countrycode FROM city ;
SELECT DISTINCT(countrycode) FROM city  ;

# 联合查询- union all
SELECT * FROM city 
WHERE countrycode IN ('CHN' ,'USA');

SELECT * FROM city WHERE countrycode='CHN'
UNION ALL
SELECT * FROM city WHERE countrycode='USA'

# 联合查询- union 
SELECT * FROM city WHERE countrycode='CHN'
UNION 
SELECT * FROM city WHERE countrycode='USA'

说明:一般情况下,我们会将 IN 或者 OR 语句 改写成 UNION (ALL),来提高性能
UNION     去重复
UNION ALL 不去重复

```





































