---
title: 16.Redis哨兵模式
date: 2020-02-22 17:39:18
tags:
    - Redis
    - 缓存
categories:
    - 中间件
    - Redis
---

类似与Mysql的主备切换，Sentinel是Redis的高可用性解决方案之一



## 16.1 主从复制的问题

- 一旦 主节点宕机，从节点 晋升成 主节点，同时需要修改 应用方 的 主节点地址，还需要命令所有 从节点去复制 新的主节点，整个过程需要人工干预。
- 主节点的写能力受到 单机的限制。
- 主节点的存储能力受到单机的限制。

## 16.2 哨兵模式作用

哨兵模式可以解决主从的第一个问题

- 集群监控：负责监控 redis master 和 slave 进程是否正常工作
- 消息通知：如果某个 redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址
- 应用透明 : 应用不需要重启改配置文件



## 16.3 哨兵模式架构

![](http://cache410.oss-cn-beijing.aliyuncs.com/sentinel.png)

- 哨兵至少需要 3 个实例，来保证自己的健壮性。(分布式quorum选举)

- 哨兵 + redis 主从的部署架构，是**不保证数据零丢失的，只能保证 redis 集群的高可用性**。

  

## 16.4 工作原理 

每个 Sentinel 节点都需要 定期执行 以下任务：

- 每个 Sentinel 以 每秒钟 一次的频率，向它所知的 主服务器、从服务器 以及其他 Sentinel 实例 发送一个 PING 命令。
- 如果一个 实例（instance）距离 最后一次 有效回复 PING 命令的时间超过 down-after-milliseconds 所指定的值，那么这个实例会被 Sentinel 标记为 主观下线。
- 如果一个 主服务器 被标记为 主观下线，那么正在 监视 这个 主服务器 的所有 Sentinel 节点，要以 每秒一次 的频率确认 主服务器 的确进入了 主观下线 状态。
- 如果一个 主服务器 被标记为 主观下线，并且有 足够数量 的 Sentinel（至少要达到 配置文件 指定的数量）在指定的 时间范围 内同意这一判断，那么这个 主服务器 被标记为 客观下线。
- 在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率，向它已知的所有 主服务器 和 从服务器 发送 INFO 命令。当一个 主服务器 被 Sentinel 标记为 客观下线 时，Sentinel 向 下线主服务器 的所有 从服务器 发送 INFO 命令的频率，会从 10 秒一次改为 每秒一次
- Sentinel 和其他 Sentinel 协商 主节点 的状态，如果 主节点 处于 SDOWN 状态，则投票自动选出新的 主节点。将剩余的 从节点 指向 新的主节点 进行 数据复制。
- 当没有足够数量的 Sentinel 同意 主服务器 下线时， 主服务器 的 客观下线状态 就会被移除。当 主服务器 重新向 Sentinel 的 PING 命令返回 有效回复 时，主服务器 的 主观下线状态 就会被移除。

### 16.4.1 sdown 和 odown 转换机制

- sdown 是主观宕机，就一个哨兵如果自己觉得一个 master 宕机了，那么就是主观宕机
- odown 是客观宕机，如果 quorum 数量的哨兵都觉得一个 master 宕机了，那么就是客观宕机

sdown 达成的条件很简单，如果一个哨兵 ping 一个 master，超过了 is-master-down-after-milliseconds 指定的毫秒数之后，就主观认为 master 宕机了；如果一个哨兵在指定时间内，收到了 quorum 数量的其它哨兵也认为那个 master 是 sdown 的，那么就认为是 odown 了。

### 16.4.2 哨兵集群的自动发现机制

哨兵互相之间的发现，是通过 redis 的 pub/sub 系统实现的，每个哨兵都会往 __sentinel__:hello 这个 channel 里发送一个消息，这时候所有其他哨兵都可以消费到这个消息，并感知到其他的哨兵的存在。

每隔两秒钟，每个哨兵都会往自己监控的某个 master+slaves 对应的 __sentinel__:hello channel 里发送一个消息，内容是自己的 host、ip 和 runid 还有对这个 master 的监控配置。

每个哨兵也会去监听自己监控的每个 master+slaves 对应的 __sentinel__:hello channel，然后去感知到同样在监听这个 master+slaves 的其他哨兵的存在。

每个哨兵还会跟其他哨兵交换对 master 的监控配置，互相进行监控配置的同步。

### 16.4.3 选举算法

如果一个 master 被认为 odown 了，而且 majority 数量的哨兵都允许主备切换，那么某个哨兵就会执行主备切换操作，此时首先要选举一个 slave 来，会考虑 slave 的一些信息：

- 跟 master 断开连接的时长
- slave 优先级
- 复制 offset
- run id

如果一个 slave 跟 master 断开连接的时间已经超过了 down-after-milliseconds 的 10 倍，外加 master 宕机的时长，那么 slave 就被认为不适合选举为 master。

接下来会对 slave 进行排序：

- 按照 slave 优先级进行排序，slave priority 越低，优先级就越高。
- 如果 slave priority 相同，那么看 replica offset，哪个 slave 复制了越多的数据，offset 越靠后，优先级就越高。
- 如果上面两个条件都相同，那么选择一个 run id 比较小的那个 slave。

### 16.4.4 configuration传播

执行切换的那个哨兵，会从要切换到的新 master（salve->master）那里得到一个 configuration epoch，这就是一个 version 号，每次切换的 `version`号都必须是唯一的。

如果第一个选举出的哨兵切换失败了，那么其他哨兵，会等待 failover-timeout 时间，然后接替继续执行切换，此时会重新获取一个新的 configuration epoch，作为新的 `version` 号。

哨兵完成切换之后，会在自己本地更新生成最新的 master 配置，然后同步给其他的哨兵，就是通过之前说的 pub/sub 消息机制。

这里之前的 version 号就很重要了，因为各种消息都是通过一个 channel 去发布和监听的，所以一个哨兵完成一次新的切换之后，新的 master 配置是跟着新的` version` 号的。其他的哨兵都是根据版本号的大小来更新自己的 master 配置的。

## 16.5 主备切换数据丢失问题

主备切换的过程，可能会导致数据丢失：

- 异步复制导致的数据丢失
- 脑裂导致的数据丢失

### 16.5.1 异步复制导致的数据丢失 

因为 master->slave 的复制是异步的，所以可能有部分数据还没复制到 slave，master 就宕机了，此时这部分数据就丢失了

### 16.5.2 脑裂导致的数据丢失

脑裂，也就是说，某个 master 所在机器突然脱离了正常的网络，跟其他 slave 机器不能连接，但是实际上 master 还运行着。此时哨兵可能就会认为 master 宕机了，然后开启选举，将其他 slave 切换成了 master。这个时候，集群里就会有两个 master ，也就是所谓的**脑裂**。

此时虽然某个 slave 被切换成了 master，但是可能 client 还没来得及切换到新的 master，还继续向旧 master 写数据。因此旧 master 再次恢复的时候，会被作为一个 slave 挂到新的 master 上去，自己的数据会清空，重新从新的 master 复制数据。而新的 master 并没有后来 client 写入的数据，因此，这部分数据也就丢失了。



### 16.5.3 数据丢失问题的解决方案

**注意：数据丢失只能说减少丢失不能完全规避丢失**

进行如下配置：

```ini
min-slaves-to-write 1
min-slaves-max-lag 10
```


表示，要求至少有 1 个 slave，数据复制和同步的延迟不能超过 10 秒。

如果说一旦所有的 slave，数据复制和同步的延迟都超过了 10 秒钟，那么这个时候，master 就不会再接收任何请求了。

- 减少异步复制数据的丢失

有了 min-slaves-max-lag 这个配置，就可以确保说，一旦 slave 复制数据和 ack 延时太长，就认为可能 master 宕机后损失的数据太多了，那么就拒绝写请求，这样可以把 master 宕机时由于部分数据未同步到 slave 导致的数据丢失降低的可控范围内。

- 减少脑裂的数据丢失

如果一个 master 出现了脑裂，跟其他 slave 丢了连接，那么上面两个配置可以确保说，如果不能继续给指定数量的 slave 发送数据，而且 slave 超过 10 秒没有给自己 ack 消息，那么就直接拒绝客户端的写请求。因此在脑裂场景下，最多就丢失 10 秒的数据

