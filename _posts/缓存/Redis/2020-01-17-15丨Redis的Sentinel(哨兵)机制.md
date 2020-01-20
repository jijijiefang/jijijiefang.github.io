---
layout:     post
title:      "Redis-15丨Redis的Sentinel(哨兵)机制"
date:       2020-01-17 23:49:41
author:     "jiefang"
header-style: text
tags:
    - Redis
---
# Redis的Sentinel(哨兵)机制
Redis Sentinel是Redis的高可用实现方案。

名词|逻辑结构|物理结构
---|---|---
主节点(master)|Redis主服务/数据库|一个独立的Redis进程
从节点(slave)|Redis从服务/数据库|一个独立的Redis进程
||
Redis数据节点|主节点和从节点|主节点和从节点的进程
Sentinel节点|监控Redis数据节点|一个独立的Sentinel进程
Sentinel节点集合|若干Sentinel节点的抽象组合|若干Sentinel节点进程
Redis Sentinel|Redis高可用实现方案|Sentinel节点集合和Redis数据节点进程
应用方|泛指一个或多个客户端|一个或多个客户端进程或者线程


## Sentinel(哨兵)由来
Redis的主从复制模式可以将主节点的数据改变同步给从节点，这样从 节点就可以起到两个作用：
- 作为主节点的一个备份，一旦主节点出了故障不可达的情况，从节点可以作为后备“顶”上来，并且保证数据尽量不丢 失（主从复制是最终一致性）。
- 从节点可以扩展主节点的读能力，一旦主节点不能支撑住大并发量的读操作，从节点可以在一定程度上帮助主节 点分担读压力。

主从复制的问题：
- 一旦主节点出现故障，需要手动将一个从节点晋升为主节点，同时需 要修改应用方的主节点地址，还需要命令其他从节点去复制新的主节点，整 个过程都需要人工干预；
- 主节点的写能力受到单机的限制；
- 主节点的存储能力受到单机的限制；

当主节点出现故障时，Redis Sentinel能自动完成故障发现和故障转移， 并通知应用方，从而实现真正的高可用。

Redis Sentinel是一个分布式架构，其中包含若干个Sentinel节点和Redis 数据节点，每个Sentinel节点会对数据节点和其余Sentinel节点进行监控，当 它发现节点不可达时，会对节点做下线标识。如果被标识的是主节点，它还 会和其他Sentinel节点进行“协商”，当大多数Sentinel节点都认为主节点不可 达时，它们会选举出一个Sentinel节点来完成自动故障转移的工作，同时会 将这个变化实时通知给Redis应用方。

![Redis主从复制与Redis Sentinel架构的区别](https://s2.ax1x.com/2020/01/17/1S03QJ.png)

### 故障转移处理逻辑
1. 主节点出现故障，此时两个从节点与主节点失去连 接，主从复制失败;
2. 每个Sentinel节点通过定期监控发现主节点出现了故障;
3. 多个Sentinel节点对主节点的故障达成一致，选举出某个sentinel节点作为领导者负责故障转移；
4. Sentinel领导者节点执行了故障转移；

### 功能
Redis Sentinel具有以下几个功能：
- **监控**：Sentinel节点会定期检测Redis数据节点、其余Sentinel节点是否 可达；
- **通知**：Sentinel节点会将故障转移的结果通知给应用方；
- **主节点故障转移**：实现从节点晋升为主节点并维护后续正确的主从关系；
- **配置提供者**：在Redis Sentinel结构中，客户端在初始化的时候连接的是Sentinel节点集合，从中获取主节点信息；

Redis Sentinel包含了若个Sentinel节点，这样的好处：
- 对于节点的故障判断是由多个Sentinel节点共同完成，这样可以有效地 防止误判；
- ·Sentinel节点集合是由若干个Sentinel节点组成的，这样即使个别Sentinel 节点不可用，整个Sentinel节点集合依然是可用；

## 原理
Redis Sentinel通过三个定时监控任务完成对各个节点发现和监控：
- 每隔10秒，每个Sentinel节点会向主节点和从节点发送info命令获取 最新的拓扑结构；
![Sentinel节点定时执行info命令](https://s2.ax1x.com/2020/01/17/1SB2u9.png)
>主节点上执行info replication：
```
# Replication
role:master
connected_slaves:2 
slave0:ip=127.0.0.1,port=6380,state=online,offset=4917,lag=1
slave1:ip=127.0.0.1,port=6381,state=online,offset=4917,lag=1
```
Sentinel节点通过对上述结果进行解析就可以找到相应的从节点。

这个定时任务的作用：

    - 通过向主节点执行info命令，获取从节点的信息；
    - 当有新的从节点加入时都可以立刻感知出来；
    - 节点不可达或者故障转移后，可以通过info命令实时更新节点拓扑信息；

- 每隔2秒，每个Sentinel节点会向Redis数据节点的`__sentinel__：hello` 频道上发送该Sentinel节点对于主节点的判断以及当前Sentinel节点的信息,同时每个Sentinel节点也会订阅该频道，来了解其他 Sentinel节点以及它们对主节点的判断。
![Sentinel节点发布和订阅__sentinel__hello频道](https://s2.ax1x.com/2020/01/17/1SDXM4.md.png)
    - 发现新的Sentinel节点：通过订阅主节点的`__sentinel__：hello`了解其他 的Sentinel节点信息，如果是新加入的Sentinel节点，将该Sentinel节点信息保 存起来，并与该Sentinel节点创建连接
    - Sentinel节点之间交换主节点的状态，作为后面客观下线以及领导者选举的依据；
- 每隔1秒，每个Sentinel节点会向主节点、从节点、其余Sentinel节点发送一条ping命令做一次心跳检测，来确认这些节点当前是否可达;
![Sentinel节点向其余节点发送ping命令](https://s2.ax1x.com/2020/01/17/1SrFzD.md.png)

## 主观下线和客观下线
### 主观下线
**主观下线**:每个Sentinel节点会每隔1秒对主节点、从节点、其他Sentinel节点发送ping命令做心跳检测，当这些节点超过`down-after-milliseconds`没有进行有效回复，Sentinel节点就会对该节点做失败判定，这个行为叫做主观下线。

![主观下线](https://s2.ax1x.com/2020/01/17/1Srz6g.md.png)

### 客观下线
**客观下线**：当Sentinel主观下线的节点是主节点时，该Sentinel节点会通过`sentinel is- master-down-by-addr`命令向其他Sentinel节点询问对主节点的判断，当超过`<quorum>`个数，Sentinel节点认为主节点确实有问题，这时该Sentinel节点会做出客观下线的决定。
![客观下线](https://s2.ax1x.com/2020/01/17/1SsunJ.png)

## Sentinel节点选举
当Sentinel节点对于主节点已经做了客观下线以后，Sentinel节点之间会做一个领导者选举的工作，选出一个Sentinel节点作为领导者进行故障转移的工作，Redis使用了Raft算法实 现领导者选举。

Redis Sentinel选举思路：
1. 每个在线的Sentinel节点都有资格成为领导者，当它确认主节点主观下线时候，会向其他Sentinel节点发送`sentinel is-master-down-by-addr`命令， 要求将自己设置为领导者。
2. 收到命令的Sentinel节点，如果没有同意过其他Sentinel节点的`sentinel is-master-down-by-addr`命令，将同意该请求，否则拒绝。
3. 如果该Sentinel节点发现自己的票数已经大于等于`max（quorum， num（sentinels）/2+1）`，那么它将成为领导者。
4. 如果此过程没有选举出领导者，将进入下一次选举。

### 示例
![image](https://s2.ax1x.com/2020/01/17/1Ssx4x.png)
1. s1（sentinel-1）最先完成了客观下线，它会向s2（sentinel-2）和s3（sentinel-3）发送`sentinel is-master-down-by-addr`命令，s2和s3同意选其为领导者;
2. s1此时已经拿到2张投票，满足了大于等于`max（quorum，num（sentinels）/2+1）=2`的条件，所以此时s1成为领导者;

## 故障转移
领导者选举出的Sentinel节点负责故障转移，具体步骤如下：
1. 在从节点列表中选出一个节点作为新的主节点，选择方法如下：
    - 过滤：“不健康”（主观下线、断线）、5秒内没有回复过Sentinel节 点ping响应、与主节点失联超过`down-after-milliseconds*`10秒；
    - 选择`slave-priority`（从节点优先级）最高的从节点列表，如果存在则 返回，不存在则继续;
    - 选择复制偏移量最大的从节点（复制的最完整），如果存在则返回，不存在则继续;
    - 选择runid最小的从节点;
2. Sentinel领导者节点会对第一步选出来的从节点执行`slaveof no one`命 令让其成为主节点;
3. Sentinel领导者节点会向剩余的从节点发送命令，让它们成为新主节点的从节点，复制规则和parallel-syncs参数有关;
4. Sentinel节点集合会将原来的主节点更新为从节点，并保持着对其关注，当其恢复后命令它去复制新的主节点;

## 总结
![image](https://s2.ax1x.com/2020/01/17/1SyU2T.md.png)
1. Redis Sentinel是Redis的高可用实现方案：故障发现、故障自动转移、配置中心、客户端通知；
2. Redis Sentinel从Redis2.8版本开始才正式生产可用，之前版本生产不可用；
3. 尽可能在不同物理机上部署Redis Sentinel所有节点；
4. Redis Sentinel中的Sentinel节点个数应该为大于等于3且最好为奇数；
5. Redis Sentinel通过三个定时任务实现了Sentinel节点对于主节点、从节点、其余Sentinel节点的监控；
6. Redis Sentinel在对节点做失败判定时分为主观下线和客观下线；