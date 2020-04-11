---
title: TCP关闭连接
date: 2020-04-11 21:02:00
tags:
    - 网络
    - TCP
categories:
    - 基础
    - 网络
---


## 1 两个关键的参数

- **time_wait**:客户端断开连接时
- **close_wait**:服务端断开连接时

## 2 time_wait

### 2.1 为什么会有time_wait

- **可靠地实现TCP全双工连接的终止。（确保最后的ACK能让被关闭方接收）**
- **允许老的重复分节在网络中消逝**

### 2.2 隐患

在socket的处于TIME_WAIT状态之后到它结束之前，**该socket所占用的本地端口号将一直无法释放**，因此服务在高并发高负载下运行一段时间后，就常常会出现做为客户端的程序**无法向服务端建立新的sockett连接的情况**

这是因为**服务方socket资源已经耗尽**。**netstat命令查看系统将会发现机器上存在大量处于TIME_WAIT状态的socket连接**



### 2.3 解决方法

- **net.ipv4.tcp_tw_reuse = 1**
  - 开启后，作为客户端时新连接**可以使用仍然处于 TIME-WAIT 状态的端口**
  -  需要把**net.ipv4.tcp_timestamps设置为1**,由于 timestamp 的存在，操作系统可以拒绝迟到的报文 
- **net.ipv4.tcp_max_tw_buckets = 262144**

  - **time_wait 状态连接的最大数量** 
  - **超出后直接关闭连**

- net.ipv4.tcp_tw_recycle = 0 
  - 开启后，同时作为客户端和服务器都可以使用 TIME-WAIT 状态的端口
  - **不安全**，无法避免报文延迟、重复等给新连接造成混乱
  - **不要使用**
- Windows下可以修改time_wait的等待时间,但Linux下不能修改



## 3 close_wait

**一般都是代码问题～**

**没有立即发送FIN，堵塞住了**

检查代码,有可能的原因:

- **Mysql没有rollback或commit**　自身经历．．．



**可以使用perf或者strace工具进行分析查看**