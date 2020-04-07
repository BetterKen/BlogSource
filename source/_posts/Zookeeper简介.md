---
title: Zookeeper简介
date: 2020-04-07 22:00:00
tags:
    - 分布式
    - zookeeper
categories:
    - 分布式
---

## 1.1 概述

Zookeeper从设计模式角度来理解：是一个基于观察者模式设计的分布式服务管理框架，它**负**
**责存储和管理大家都关心的数据**，然后**接受观察者的注册**，一旦这些数据的状态发生变化 ，
ookeeper就将**负责通知已经在Zookeeper上注册的那些观察者**做出相应的反应。

## 1.2 特点

![](http://dist415.oss-cn-beijing.aliyuncs.com/zookeeper.png)

1. Zookeeper：一个领导者（Leader），多个跟随者（Follower）组成的集群。
2. 集群中只要有半数以上节点存活，Zookeeper集群就能正常服务。
3. 全局数据一致：每个Server保存一份相同的数据副本，Client无论连接到哪个Server，数据都是一致的。
4. 更新请求顺序进行，来自同一个Client的更新请求按其发送顺序依次执行。
5. 数据更新原子性，一次数据更新要么成功，要么失败。
6. 实时性，在一定时间范围内，Client能读到最新数据。

## 1.3 数据结构

ZooKeeper数据模型的结构与**Unix文件系统很类似**，整体上可以看作是一棵树，每个节点称做一
个ZNode。每一个ZNode默认能够存储**1MB**的数据，每个ZNode都可以**通过其路径唯一标识**。

![](http://dist415.oss-cn-beijing.aliyuncs.com/zkstruct.png)

## 1.4 应用场景

- 分布式协调
- 分布式锁
- 元数据/配置信息管理
- HA高可用性



### 1.4.1 分布式协调

这个其实是 zookeeper 很经典的一个用法，简单来说，就好比，你 A 系统发送个请求到 mq，然后 B 系统消息消费之后处理了。那 A 系统如何知道 B 系统的处理结果？用 zookeeper 就可以实现分布式系统之间的协调工作。A 系统发送请求之后可以在 zookeeper 上对某个节点的值**注册个监听器**，一旦 B 系统处理完了就修改 zookeeper 那个节点的值，A 系统立马就可以收到通知，完美解决。

![](http://dist415.oss-cn-beijing.aliyuncs.com/zookeeper-distributed-coordination.png)

### 1.4.2 分布式锁

举个栗子。对某一个数据连续发出两个修改操作，两台机器同时收到了请求，但是只能一台机器先执行完另外一个机器再执行。那么此时就可以使用 zookeeper 分布式锁，一个机器接收到了请求之后先获取 zookeeper 上的一把分布式锁，就是可以去**创建一个 znode**，接着执行操作；然后另外一个机器也**尝试去创建那个 znode**，结果发现自己创建不了，因为被别人创建了，那只能等着，等第一个机器执行完了自己再执行。

![](http://dist415.oss-cn-beijing.aliyuncs.com/zookeeper-distributed-lock-demo.png)

### 1.4.3 元数据/配置信息管理

zookeeper 可以用作很多系统的配置信息的管理，比如 kafka、storm 等等很多分布式系统都会选用 zookeeper 来做一些元数据、配置信息的管理，包括 dubbo 注册中心也支持 zookeeper 

![](http://dist415.oss-cn-beijing.aliyuncs.com/zookeeper-meta-data-manage.png)

### 1.4.4 HA高可用性

这个应该是很常见的，比如 hadoop、hdfs、yarn 等很多大数据系统，都选择基于 zookeeper 来开发 HA 高可用机制，就是一个**重要进程一般会做主备两个**，主进程挂了立马通过 zookeeper 感知到切换到备用进程。

![](http://dist415.oss-cn-beijing.aliyuncs.com/zookeeper-active-standby.png)

## 1.5 节点类型

分为四类:

- **持久化目录节点**:客户端与Zookeeper断开连接后，该节点依旧存在
- **持久化顺序编号目录节点**:客户端与Zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号
- **临时目录节点**:客户端与Zookeeper断开连接后，该节点被删除
- **临时顺序编号目录节点**:客户端 与 Zookeeper 断开连接后 ， 该 节 点 被 删 除 ， 只 是
  Zookeeper给该节点名称进行顺序编号

![](http://dist415.oss-cn-beijing.aliyuncs.com/nodeclass.png)

## 1.6 监听器原理

### 1.6.1 监听过程详解

1. 首先要有一个main()线程
2. 在main线程中创建Zookeeper客户端，这时就会创建两个线程，一个负责网络连接通信（connet），一个负责监听（listener）
3. 通过connect线程将注册的监听事件发送给Zookeeper。
4. 在Zookeeper的注册监听器列表中将注册的监听事件添加到列表中。
5. Zookeeper监听到有数据或路径变化，就会将这个消息发送给listener线程。 
6. listener线程内部调用了process()方法。

![](http://dist415.oss-cn-beijing.aliyuncs.com/zkwatch.png)



### 1.6.2 常见的监听

- **监听节点数据的变化**: get path [watch] 
- **监听子节点增减的变化**: ls path [watch]



## 1.7 选举机制

### 1.7.1 核心机制

- **半数机制：集群中半数以上(不包括半数)机器存活，集群可用**。所以 Zookeeper 适合安装奇数台服务器。

- Zookeeper 虽然在配置文件中并没有指定 Master 和 Slave。但是，Zookeeper 工作时，是有**一个节点为 Leader，其他则为 Follower**，Leader 是通过**内部的选举机制临时产生的**。

### 1.7.2 选举过程demo

假设有五台服务器组成的 Zookeeper 集群，它们的 id 从 1-5，同时它们都是最新启动的，
也就是没有历史数据，在存放数据量这一点上，都是一样的。假设这些服务器依序启动，来
看看会发生什么，如下图所示:

![](http://dist415.oss-cn-beijing.aliyuncs.com/zkvote.png)

1. 服务器 1 启动，发起一次选举。服务器 1 投自己一票。此时服务器 1 票数一票，不够半数以上（3 票），选举无法完成，服务器 1 状态保持为 LOOKING；
2. 服务器 2 启动，再发起一次选举。服务器 1 和 2 分别投自己一票并交换选票信息：此时服务器 1 发现服务器 2 的 ID 比自己目前投票推举的（服务器 1）大，更改选票为推举服务器 2。此时服务器 1 票数 0 票，服务器 2 票数 2 票，没有半数以上结果，选举无法完成，服务器 1，2 状态保持 LOOKING
3. 服务器 3 启动，发起一次选举。此时服务器 1 和 2 都会更改选票为服务器 3。此次投票结果：服务器 1 为 0 票，服务器 2 为 0 票，服务器 3 为 3 票。此时服务器 3 的票数已经超过半数，服务器 3 当选 Leader。服务器 1，2 更改状态为 FOLLOWING，服务器 3 更改状态为 LEADING；
4. 服务器 4 启动，发起一次选举。此时服务器 1，2，3 已经不是 LOOKING 状态，不会更改选票信息。交换选票信息结果：服务器 3 为 3 票，服务器 4 为 1 票。此时服务器 4服从多数，更改选票信息为服务器 3，并更改状态为 FOLLOWING；
5. 服务器 5 启动，同 4 一样当小弟。



## 1.8 写数据流程

![](http://dist415.oss-cn-beijing.aliyuncs.com/zkwrite.png)

