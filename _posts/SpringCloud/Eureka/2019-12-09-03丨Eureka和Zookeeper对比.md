---
layout:     post
title:      "Eureka-03丨Eureka和Zookeeper对比"
date:       2019-12-09 14:35:06
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# Eureka和Zookeeper对比

## CAP原则

CAP原则又称CAP定理，指的是在分布式系统的设计中，没有一种设计可以同时满足 `Consistency（一致性）`、 `Availability（可用性）`、`Partition tolerance（分区容错性）`3个特性，这三者不可得兼。

![image](https://s2.ax1x.com/2019/12/06/QJs4iQ.png)

- `Consistency`:一致性，在分布式系统中的所有数据备份，在同一时刻是否同样的值。
- `Availability`:可用性，指系统提供的服务必须一直处于可用的状态，每次请求都能获取到非错的响应,但是不保证获取的数据为最新数据。
- `Partition tolerance`:分区容错性，分布式系统在遇到任何网络分区故障的时候，仍然能够对外提供满足一致性和可用性的服务，除非整个网络环境都发生了故障。

组合：
- **AC**:满足原子和可用，放弃分区容错。说白了，就是一个整体的应用。
- **CP**:满足原子和分区容错，也就是说，要放弃可用。当系统被分区，为了保证原子性，必须放弃可用性，让服务停用。
- **AP**:满足可用性和分区容错，当出现分区，同时为了保证可用性，必须让节点继续对外服务，这样必然导致失去原子性。

## Eureka和Zookeeper

### Zookeeper
Zookeeper是基于**CP**来设计的，即任何时刻对Zookeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性，但是它不能保证每次服务请求的可用性。从实际情况来分析，在使用Zookeeper获取服务列表时，**如果zookeeper正在选主，或者Zookeeper集群中半数以上机器不可用**，那么将无法获得数据。所以说，Zookeeper不能保证服务可用性。

### Eureka
而Spring Cloud Netflix在设计Eureka时遵守的就是**AP**原则。Eureka Server也可以运行多个实例来构建集群，解决单点问题，但不同于ZooKeeper的选举leader的过程，Eureka Server采用的是**Peer to Peer**对等通信。这是一种去中心化的架构，无master/slave区分，每一个Peer都是对等的。在这种架构中，节点通过彼此互相注册来提高可用性，每个节点需要添加一个或多个有效的serviceUrl指向其他节点。每个节点都可被视为其他节点的副本。

如果某台Eureka Server宕机，Eureka Client的请求会自动切换到新的Eureka Server节点，当宕机的服务器重新恢复后，Eureka会再次将其纳入到服务器集群管理之中。当节点开始接受客户端请求时，所有的操作都会进行replicateToPeer（节点间复制）操作，将请求复制到其他Eureka Server当前所知的所有节点中。

一个新的Eureka Server节点启动后，会首先尝试从邻近节点获取所有实例注册表信息，完成初始化。Eureka Server通过getEurekaServiceUrls()方法获取所有的节点，并且会通过心跳续约的方式定期更新。默认配置下，如果Eureka Server在一定时间内没有接收到某个服务实例的心跳，Eureka Server将会注销该实例（默认为90秒，通过eureka.instance.lease-expiration-duration-in-seconds配置）。当Eureka Server节点在短时间内丢失过多的心跳时（比如发生了网络分区故障），那么这个节点就会进入自我保护模式。


## 总结
ZooKeeper基于CP，不保证高可用，如果zookeeper正在选主，或者Zookeeper集群中半数以上机器不可用，那么将无法获得数据。Eureka基于AP，能保证高可用，即使所有机器都挂了，也能拿到本地缓存的数据。作为注册中心，其实配置是不经常变动的，只有发版和机器出故障时会变。对于不经常变动的配置来说，CP是不合适的，而AP在遇到问题时可以用牺牲一致性来保证可用性，既返回旧数据，缓存数据。所以理论上Eureka是更适合作注册中心。