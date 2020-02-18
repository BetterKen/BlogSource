---
title: 10.MysqlInnodb存储引擎
date: 2020-02-18 20:15:00
tags:
    - Mysql
categories:
    - 数据库
    - Mysql
---



## 10.1 关键特性

- B-tree索引+聚簇索引
- 数据缓存与加密
- 外键支持
- 行级锁
- 事务和MVCC
- 复制和备份恢复

![](http://mysql317.oss-cn-beijing.aliyuncs.com/InnoDBStorageEngineFeatures.png)

## 10.2 Innodb逻辑结构图

![](http://mysql317.oss-cn-beijing.aliyuncs.com/innodb-architecture5.7.png)



## 10.3 Innodb内存结构

主要包括以下几部分:

- Buffer Pool
- Change Buffer
- AHI
- Log Buffer

### 10.3.1 Buffer Pool

#### 10.3.1.1 作用

前面的文章Mysql索引介绍中,我们知道Innodb是以页为储存单位来保存数据的,`读取和处理数据的时候同样需要已页为单位将数据加载到内存中` 

如果没有Buffer Pool会进行频繁的磁盘IO,这样会大大拉低Mysql的性能,因此Mysql使用Buffer Pool来做存储数据和索引的缓存,容许在内存中直接操作表数据,提高处理速度

>Buffer Pool 的功能就是 缓存“页” ，减少磁盘IO,提高读写效率。

#### 10.3.1.2 Buffer Pool结构

![](http://mysql317.oss-cn-beijing.aliyuncs.com/buffer_pool1.png)

![](http://mysql317.oss-cn-beijing.aliyuncs.com/bufferpoolself.png)



Buffer Poll是由一个个Instance组成,每个instance都有自己的锁，信号量，物理块(Buffer chunks)以及逻辑链表(下面的各种List)，即`各个instance之间没有竞争关系，可以并发读取与写入`。所有instance的物理块(Buffer chunks)在数据库启动的时候被分配，直到数据库关闭内存才予以释放

```
innodb_buffer_pool_instances = innodb_buffer_pool_size/innodb_buffer_pool_instance
Instance实例数目=BufferPool总大小/每个Instance的大小
【注意】当innodb_buffer_pool_size小于1GB时候，innodb_buffer_pool_instances被重置为1，主要是防止有太多小的instance从而导致性能问题
```

每个Buffer Pool Instance有一个page hash链表，通过它，使用space_id和page_no就能快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。

注意这个hash表不是InnoDB的自适应哈希(AHI)，自适应哈希是为了减少Btree的扫描，而page hash是为了避免扫描LRU List。

`Buffer Pool会通过三种Page和链表来管理这些经常访问的数据，保证热数据不被置换出Buffer Pool`

#### 10.3.1.3 Page的分类



![](http://mysql317.oss-cn-beijing.aliyuncs.com/pageclass.png)

#### 10.3.1.4 链表的分类

链表节点是数据页的控制体(控制体中有指针指向真正的数据页)，链表中的所有节点都有同一的属性，引入其的目的是方便管理

![](http://mysql317.oss-cn-beijing.aliyuncs.com/listclass.png)





- Free List

  Free 链表 存放的是空闲页面，初始化的时候申请一定数量的页面

  在执行SQL的过程中，每次成功load 页面到内存后，会判断Free 链表的页面是否够用。如果不够用的话，就flush LRU 链表和Flush 链表来释放空闲页。如果够用，就从Free 链表里面删除对应的页面，在LRU 链表增加页面，保持总数不变。

- LRU List

  默认情况下：

  1. Old 链表占整个LRU 链表的比例是3/8。该比例由`innodb_old_blocks_pct`控制，默认值是37（3/8*100）。该值取值范围为5~95，为全局动态变量。

  2. 当新的页被读取到Buffer Pool里面的时候，和`传统的LRU算法插入到LRU链表头部不同，Innodb LRU算法是将新的页面插入到Yong 链表的尾部和Old 链表的头部中间的位置，这个位置叫做Mid Point`，如下图所示

     ![](http://mysql317.oss-cn-beijing.aliyuncs.com/lrulist.png)

     

     

  3. 频繁访问一个Buffer Pool的页面，会促使页面往Young链表的头部移动。如果一个Page在被读到Buffer Pool后很快就被访问(需要超过innodb_old_block_time)，那么该Page会往Young List的头部移动，但是如果一个页面是通过预读的方式读到Buffer Pool，且之后短时间内没有被访问，那么很可能在下次访问之前就被移动到Old List的尾部，而被驱逐了。

  4. 随着数据库的持续运行，新的页面被不断的插入到LRU链表的Mid Point，Old 链表里的页面会逐渐的被移动Old链表的尾部。同时，当经常被访问的页面移动到LRU链表头部的时候，那些没有被访问的页面会逐渐的被移动到链表的尾部。最终，位于Old 链表尾部的页面将被驱逐。

  5. 如果一个数据页已经处于Young 链表，当它再次被访问的时候，`只有当其处于Young 链表长度的1/4(大约值)之后，才会被移动到Young 链表的头部`。这样做的目的是减少对LRU 链表的修改，因为LRU 链表的目标是保证经常被访问的数据页不会被驱逐出去。

  6. innodb_old_blocks_time 控制的Old 链表头部页面的转移策略。`该Page需要在Old 链表停留超过innodb_old_blocks_time 时间，之后再次被访问，才会移动到Young 链表`。这么操作是避免Young 链表被那些只在innodb_old_blocks_time时间间隔内频繁访问，之后就不被访问的页面塞满，从而有效的保护Young 链表。

  7. 在全表扫描或者全索引扫描的时候，Innodb会将大量的页面写入LRU 链表的Mid Point位置，并且只在短时间内访问几次之后就不再访问了。设置innodb_old_blocks_time的时间窗口可以有效的保护Young List，保证了真正的频繁访问的页面不被驱逐。`innodb_old_blocks_time 单位是毫秒，默认值是1000，即１秒。调大该值提高了从Old链表移动到Young链表的难度，会促使更多页面被移动到Old 链表，老化，从而被驱逐`

  8. 当扫描的表很大，Buffer Pool都放不下时，可以将innodb_old_blocks_pct设置为较小的值，这样只读取一次的数据页就不会占据大部分的Buffer Pool。例如，设置innodb_old_blocks_pct = 5，会将仅读取一次的数据页在Buffer Pool的占用限制为5％。当经常扫描一些小表时，这些页面在Buffer Pool移动的开销较小，我们可以适当的调大innodb_old_blocks_pct，例如设置innodb_old_blocks_pct = 50

  

  在SHOW ENGINE INNODB STATUS 里面提供了Buffer Pool一些监控指标，有几个我们需要关注一下：

  ```mysql
  =====================================
  mysql> show engine innnodb status\G
  ……
  Pages made young 0, not young 0
  0.00 youngs/s, 0.00 non-youngs/s
  ======================================
  数据页从冷到热，称为young；not young就是数据在没有成为热数据情况下就被刷走的量(累计值)。
  ```

  

  | 指标             | 场景                           | 处理方法                                                     | 作用                                                         |
  | :--------------- | :----------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
  | youngs/s很小     | 都是一些小事务，没有大表全扫描 | 调大innodb_old_blocks_pct，减小innodb_old_blocks_time        | 使Old List 的长度更长，到Old List 的尾部消耗的时间会更久，提升下一次访问到Old List里面的页面的可能性 |
  | youngs/s很大     |                                | 可以调小innodb_old_blocks_pct，同时调大innodb_old_blocks_time | 保护热数据                                                   |
  | non-youngs/s很大 | 大量的全表扫描                 | 可以调小innodb_old_blocks_pct，同时调大innodb_old_blocks_time | 保护young list                                               |
  | non-youngs/s不大 | 存在大量全表扫描               | 调大innodb_old_blocks_time                                   | 使得这些短时间频繁访问的页面保留在Old 链表里面               |

  每隔1秒钟，Page Cleaner线程执行LRU List Flush的操作，来释放足够的Free Page。innodb_lru_scan_depth 变量控制每个Buffer Pool实例每次扫描LRU List的长度，来寻找对应的脏页，执行Flush操作。

- Flush List

    1. Flush 链表里面保存的都是脏页，也会存在于LRU 链表。
    2. Flush 链表是按照oldest_modification排序，值大的在头部，值小的在尾部
    3. 当有页面访被修改的时候，使用mini-transaction，对应的page进入Flush 链表
    4. 如果当前页面已经是脏页，就不需要再次加入Flush list，否则是第一次修改，需要加入Flush 链表
    5. 当Page Cleaner线程执行flush操作的时候，从尾部开始scan，将一定的脏页写入磁盘，推进检查点，减少recover的时间



> LRU链表和FLUSH链表的区别 

![](http://mysql317.oss-cn-beijing.aliyuncs.com/lru-flush-diff.png)



### 10.3.2 Change Buffer

#### 10.3.2.1 作用

Change buffer的主要目的是将对`二级索引`的数据操作缓存下来，以此减少二级索引的随机IO，并达到操作合并的效果。

对表执行 INSERT，UPDATE和 DELETE操作时， 索引列的值（尤其是secondary keys的值）通常按未排序顺序排列，需要大量I / O才能使二级索引更新。Change Buffer会缓存这个更新当相关页面不在Buffer Pool中，从而磁盘上的相关页面不会立即被读避免了昂贵的I / O操作



#### 10.3.2.2 参数

- 参数：innodb_change_buffer_max_size

  介绍：配置写缓冲的大小，占整个缓冲池的比例，默认值是25%，最大值是50%。

  `写多读少的业务，才需要调大这个值，读多写少的业务，25%其实也多了`

- 参数：innodb_change_buffering
  介绍：配置哪些写操作启用写缓冲，可以设置成all/none/inserts/deletes等

```mysql
mysql> select @@innodb_change_buffer_max_size;
+---------------------------------+
| @@innodb_change_buffer_max_size |
+---------------------------------+
|                              25 |
+---------------------------------+
1 row in set (0.00 sec)

mysql> select @@innodb_change_buffering;
+---------------------------+
| @@innodb_change_buffering |
+---------------------------+
| all                       |
+---------------------------+
1 row in set (0.00 sec)


```



### 10.3.3 Adaptive Hash Index(AHI)

哈希(hash)是一种非常快的查找方法,在一般情况下这种查找的时间复杂度为O(1),即一般仅需要一次查找就能定位数据。而B+树的查找次数,取决于B+树的高度,在生产环境中,B+树的高度一般为3-4层,故需要3-4次的查询。

InnoDB存储引擎会监控对表上各索引页的查询。如果观察到建立哈希索引可以带来速度提升,则建立哈希索引,称之为`自适应哈希索引(Adaptive Hash Index,AHI)` AHI是通过缓冲池的B+树页构造而来,因此建立的速度很快,而且不需要对整张表构建哈希索引。 InnoDB存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。

我们可以认为：`AHI是建立在B+Tree数索引上的索引`,是Innodb内部用来自身提高查询速度的自优化功能

 

### 10.3.4 Log Buffer

#### 10.3.4.1 功能

`负责redo日志的缓冲`

redo log日志作用后面的文章将会介绍

#### 10.3.4.2 刷盘时机

- redo log buffer 空间不足的时候
- 事务提交的时候
- 后台线程在不停的刷
- 服务正常关闭的时候
- checkpoint的时候

#### 10.3.4.3 关键参数

- innodb_flush_log_at_trx_commit

```mysql
mysql> select @@innodb_flush_log_at_trx_commit;
+----------------------------------+
| @@innodb_flush_log_at_trx_commit |
+----------------------------------+
|                                1 |
+----------------------------------+
1 row in set (0.00 sec)
作用:主要控制了innodb将log buffer中的数据写入日志文件并flush磁盘的时间点

取值分别为0、1、2三个

参数说明:

1.每次事物的提交都会引起日志文件写入、flush磁盘的操作，确保了事务的ACID；flush  到操作系统的文件系统缓存,fsync到物理磁盘.
0.表示当事务提交时，不做日志写入操作，而是每秒钟将log buffer中的数据写入文件系统缓存并且秒fsync磁盘一次；
2.每次事务提交引起写入文件系统缓存,但每秒钟完成一次fsync磁盘操作。

```

- Innodb_flush_method

```mysql
mysql> select @@Innodb_flush_method;
+-----------------------+
| @@Innodb_flush_method |
+-----------------------+
| NULL                  |
+-----------------------+
1 row in set (0.00 sec)

```

可选模式:

- FSYNC(默认值):

  - buffer pool的数据写磁盘时,要先经历os cache,然后再写到磁盘
  - log buffer的数据写磁盘时,要先经历os cache,然后再写到磁盘

- O_DSYNC:

  - buffer pool的数据写磁盘时,要先经历os cache,然后再写到磁盘
  - log buffer 的数据写磁盘时,直接写到写到磁盘,跨过os cache

- O_DIRECT：

  - buffer pool的数据写磁盘时,直接写到写到磁盘,跨过os cache

  - log buffer的数据写磁盘时,要先经历os cache,然后再写到磁盘

建议使用:O_DIRECT+固态硬盘



## 10.4 Innodb物理存储结构

​    主要包含以下几部分:

- 系统表空间 System Tablespace
- Undo日志
- Redo日志
- 临时表空间
- 用户表空间
- ib_buffer_pool

### 10.4.1 System Tablespace

5.7版本的Mysql系统表空间用来存储以下信息:

- Innodb的数据字典
- double buffer
- change buffer
- undo log

5.6版本的Mysql系统表空间还存储是临时表数据

8.8版本的Mysql系统表空间取消存储数据字典信息并将undo log独立了出来

Mysql在慢慢瘦身系统表空间,把比较关键的数据独立出来了

#### 10.4.1.1 ibdata文件

默认情况下,系统表空间在磁盘上的文件名为`ibdata1`,存在`/data`文件夹下

我们可以通过`innodb_data_file_path `参数修改系统表空间文件的数量

```mysql
innodb_data_file_path=ibdata1:1G:ibdata2:1G:autoextend
含义:有多个ibdata 第一个名称为ibdata1大小1G 第二个名称为ibdata2大小1G
autoextend意思是以此类推的生成ibdata
```

`innodb_data_file_path`参数通常要在初始化的时候加入到my.cnf文件下配置好

如果是在已运行的数据库上扩展多个ibdata文件,在设置`innodb_data_file_path`参数时,已有的ibdata1文件大小应该和磁盘上真正运行的ibdata1大小一致,而不是随便指定

#### 10.4.1.2 缩小系统表空间大小

我们不能从系统表空间中删除数据,如果想减小系统表空间的大小我们只能做如下操作:

1. mysqldump数据
2. 关闭服务
3. 删除所有 .ibd文件,ibdata文件ib_log文件
4. 删除所有.frm文件
5. 配置新系统表空间
6. 重启服务
7. 导入数据



### 10.4.2 Redo log

Redo log　重做日志

#### 10.4.2.1 功能

用户存储Mysql在做修改类操作时的数据页变化过程和版本号(LSN),属于物理日志

默认用两个文件存储redo log,是循环覆盖使用的

#### 10.4.2.2 文件位置

一般也是存储在`/data`目录下,文件名为ib_logfile0 和 ib_logfile1

#### 10.4.2.3 控制参数

```ini
innodb_log_file_size=50331648    #设置文件大小
innodb_log_files_in_group=2      #设置文件个数
innodb_log_group_home_dir=./     #设置存储位置
```



### 10.4.3 Undo log

Undo log　回滚日志

#### 10.4.3.1 功能

用来存储回滚日志,记录每次操作的反操作,属于逻辑日志

1. 使用快照功能,提供Innodb多版本并发读写
2. 通过记录的反操作,提供回滚功能

#### 10.4.3.2 文件位置

ibdataN里和ibtmp1里

#### 10.4.3.3 控制参数

```ini
 innodb_rollback_segments=128 #回滚段的个数
```



