---
layout:     post
title:      "RocketMQ-02丨基于DLedger技术的Broker主从同步原理"
date:       2021-03-08 20:53:40
author:     "jiefang"
header-style: text
tags:
    - RocketMQ
---
# 基于DLedger技术的Broker主从同步原理

![](https://s3.ax1x.com/2021/03/04/6V34yQ.png)

一条数据就会在三个Broker上有三份副本，如果Leader Broker宕机，那么就直接让其他的Follower Broker自动切换为新的Leader Broker。

## DLedger基于Raft协议选举

这需要发起一轮一轮的投票，通过三台机器互相投票选出来一个人作为Leader。

简单来说，三台Broker机器启动的时候，他们都会投票自己作为Leader，然后把这个投票发送给其他Broker。

我们举一个例子，Broker01是投票给自己的，Broker02是投票给自己的，Broker03是投票给自己的，他们都把自己的投票发送给了别人。

此时在第一轮选举中，Broker01会收到别人的投票，他发现自己是投票给自己，但是Broker02投票给Broker02自己，Broker03投票给Broker03自己，似乎每个人都很自私，都在投票给自己，所以第一轮选举是失败的。

接着每个人会进入一个随机时间的休眠，比如说Broker01休眠3秒，Broker02休眠5秒，Broker03休眠4秒。

此时Broker01必然是先苏醒过来的，他苏醒过来之后，直接会继续尝试投票给自己，并且发送自己的选票给别人。

接着Broker03休眠4秒后苏醒过来，他发现Broker01已经发送来了一个选票是投给Broker01自己的，此时他自己因为没投票，所以会尊重别人的选择，就直接把票投给Broker01了，同时把自己的投票发送给别人。

接着Broker02苏醒了，他收到了Broker01投票给Broker01自己，收到了Broker03也投票给了Broker01，那么他此时自己是没投票的，直接就会尊重别人的选择，直接就投票给Broker01，并且把自己的投票发送给别人。

此时所有人都会收到三张投票，都是投给Broker01的，那么Broker01就会当选为Leader。

其实只要有（3台机器 / 2） + 1个人投票给某个人，就会选举他当Leader，这个（机器数量 / 2） + 1就是大多数的意思。

这就是Raft协议中选举leader算法的简单描述，简单来说，他确保有人可以成为Leader的核心机制就是一轮选举不出来Leader的话，就让大家随机休眠一下，先苏醒过来的人会投票给自己，其他人苏醒过后发现自己收到选票了，就会直接投票给那个人。

## **基于Raft协议进行多副本同步**

**数据同步会分为两个阶段，一个是uncommitted阶段，一个是commited阶段**

首先Leader Broker上的DLedger收到一条数据之后，会标记为**uncommitted状态**，然后他会通过自己的DLedgerServer组件把这个uncommitted数据发送给Follower Broker的DLedgerServer。

接着Follower Broker的DLedgerServer收到uncommitted消息之后，必须返回一个ack给Leader Broker的DLedgerServer，然后如果Leader Broker收到超过半数的Follower Broker返回ack之后，就会将消息标记为**committed状态**。

然后Leader Broker上的DLedgerServer就会发送commited消息给Follower Broker机器的DLedgerServer，让他们也把消息标记为**comitted状态**。

这个就是基于Raft协议实现的两阶段完成的数据同步机制。