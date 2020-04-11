---
title: OSI模型
date: 2020-04-11 21:00:00
tags:
    - 网络
    - TCP
categories:
    - 基础
    - 网络
---

开放系统互连参考模型 (Open System Interconnect 简称OSI）是国际标准化组织(ISO)和国际电报电话咨询委员会(CCITT)联合制定的开放系统互连参考模型，为开放式互连信息系统提供了一种功能结构的框架。

使两个不同的系统能够通信,而不需要改变底层的硬件或软件逻辑。

`OSI是一种理想化的架构！是一个概念模型不是协议！`
![](http://base422.oss-cn-beijing.aliyuncs.com/netosi.png)

## 1 每层作用



- 应用层Application

为应用程序提供服务并规定应用程序中通信相关的细节。包括文件传输，电子邮件，远程登陆等协议。

- 表示层Presntation

主要负责数据格式的转换。将设备固有数据格式转换成网络标准传输格式。

- 会话层Session

负责建立和断开通信连接，以及数据的分割等数据传输相关的管理。

- 传输层Transport

起可靠的传输作用。

- 网络层Network

将数据传输到目标地址，主要负责寻址和路由选择。

- 数据链路层Datalink

将0，1序列划分成具有意义的数据帧传给对端。

- 物理层Physical

负责0，1比特流与电压高低之间的转换。

![](http://base422.oss-cn-beijing.aliyuncs.com/netdetail.png)
## 2 实际模型 

OSI只是一个理想的模型，将通信分为了七层，但在实际应用中，是将通信分成了五层（也有分成四层的，将数据链路层和物理层合为了一层）

![](http://base422.oss-cn-beijing.aliyuncs.com/netfun.png)

## 3 封装与对等通信

![](http://base422.oss-cn-beijing.aliyuncs.com/netpack.png)

实际数据的传输是一个逐层封装的过程，到达另一端后，再逐层解封装。
两台主机之间的通信都是对等层之间的通信。



>封装
上层的数据被封装在下层中，在第N层的分组的数据部分是第N+1层的整个分组。

除链路层在数据的头尾进行封装外，应用层，运输层和网络层都只是在数据的头部进行封装。
