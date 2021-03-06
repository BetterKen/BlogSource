---
title: 1.Redis简介
date: 2020-02-18 13:15:15
tags:
    - Redis
    - 缓存
categories:
    - 中间件
    - Redis
---



## 1.1 Redis特点 

- Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
- Redis支持数据的备份，即master-slave模式的数据备份
- Redis有原生的集群模式
- Redis是单线程模式



## 1.2 与Memcache对比

|   名称    |   持久化   | 数据类型 | 集群支持                                                    |
| :-------: | :--------: | :------: | ----------------------------------------------------------- |
|   Redis   |  可持久化  |   五种   | 原生支持cluster模式                                         |
| Memcached | 不可持久化 |   一种   | 没有原生的集群模式,需要依靠客户端来实现往集群中分片写入数据 |



## 1.3 Redis线程模型

redis 基于 `reactor 模式`开发了网络事件处理器，这个处理器叫做文件事件处理器，file event handler，这个文件事件处理器是单线程的，所以redis 是单线程的模型，采用 io多路复用机制同时监听多个 socket,根据socket上的事件来选择对应的事件处理器来处理这个事件

文件事件处理器的结构包含 4 个部分：

- 多个 Socket
- IO 多路复用程序
- 文件事件分派器
- 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）

多个 Socket 可能会并发产生不同的操作，每个操作对应不同的文件事件，但是 IO 多路复用程序会监听多个 Socket，会将 Socket 产生的事件放入队列中排队，事件分派器每次从队列中取出一个事件，把该事件交给对应的事件处理器进行处理。

![](http://cache410.oss-cn-beijing.aliyuncs.com/even.png)





## 1.4 Redis为什么效率高

- 纯内存操作
- 核心是基于非堵塞的IO多路复用机制
- 单线程反而避免了多线程的频繁上下文切换问题

## 1.5 Redis 数据结构



![](http://cache410.oss-cn-beijing.aliyuncs.com/datastruct.png)



​	



























