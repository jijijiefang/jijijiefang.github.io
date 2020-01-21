---
layout:     post
title:      "Redis-16丨Redis Cluster（集群）原理"
date:       2020/1/21 14:26
author:     "jiefang"
header-style: text
tags:
    - Redis
---
# Redis Cluster（集群）原理
Redis Cluster是Redis的分布式解决方案，在3.0版本正式推出，有效地解
决了Redis分布式方面的需求。节点之间通过去中心化的方式提供了完整的sharding(数据分片)、replication(复制机制、Cluster具备感知准备的能力)、failover解决方案。
## 拓扑结构
一个Redis Cluster由多个Redis节点组成。不同的节点组服务的数据无交集，每个节点对应数据sharding的一个分片。节点组内部分为主备2类，对应master和slave。两者数据准实时一致，通过异步化的主备复制机制保证。一个节点组有且仅有一个master，同时有0到多个slave。只有master对外提供写服务，读服务可由master/slave提供。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/97a5b75d4bb069530cfd8f7e873043de?showdoc=.jpg)
Redis Cluster总共有16384个slot，每一个节点负责一部分slot。

Redis Cluster中所有的几点之间两两通过Redis Cluster Bus交互，主要交互以下关键信息：
- 数据分片(slot)和节点的对应关系；
- 集群中每个节点可用状态；
- 集群结构发生变更时，通过一定的协议对配置信息达成一致。数据分片的迁移、故障发生时的主备切换决策、单点- master的发现和其发生主备关系的变更等场景均会导致集群结构变化；
- publish和subscribe(发布/订阅)功能在cluster版的内容实现所需要交互的信息；

## 节点通信
### 通信流程
Redis集群采用P2P的Gossip（流言）协议，Gossip协议工作原理就是节点彼此不断通信交换信息，一段时间后所有的节点都会知道集群完整的信息，这种方式类似流言传播。
通信过程：
- 集群中的每个节点都会单独开辟一个TCP通道，用于节点之间彼此通信，通信端口号在基础端口上加10000；
- 每个节点在固定周期内通过特定规则选择几个节点发送ping消息；
- 接收到ping消息的节点用pong消息作为响应；
### Gossip消息
Gossip协议的主要职责就是信息交换。常用的Gossip消息可分为：**ping消息、pong消息、meet消息、fail消息**
等。
- **meet消息**：用于通知新节点加入。消息发送者通知接收者加入到当前集群，meet消息通信正常完成后，接收节点会加入到集群中并进行周期性的ping、pong消息交换。
- **ping消息**：集群内交换最频繁的消息，集群内每个节点每秒向多个其他节点发送ping消息，用于检测节点是否在线和交换彼此状态信息。ping消息发送封装了自身节点和部分其他节点的状态数据。
- **pong消息**：当接收到ping、meet消息时，作为响应消息回复给发送方确认消息正常通信。pong消息内部封装了自身状态数据。
- **fail消息**：当节点判定集群内另一个节点下线时，会向集群内广播一个fail消息，其他节点接收到fail消息之后把对应节点更新为下线状态。

#### 消息格式
消息格式划分为：消息头和消息体。
##### 消息头
消息头包含发送节点自身状态数据，接收节点根据消息头就可以获取到发送节点的相关数据。
```
typedef struct {
	char sig[4]; /* 信号标示 */
	uint32_t totlen; /* 消息总长度 */
	uint16_t ver; /* 协议版本*/
	uint16_t type; /* 消息类型,用于区分meet,ping,pong等消息 */
	uint16_t count; /* 消息体包含的节点数量，仅用于meet,ping,ping消息类型*/
	uint64_t currentEpoch; /* 当前发送节点的配置纪元 */
	uint64_t configEpoch; /* 主节点/从节点的主节点配置纪元 */
	uint64_t offset; /* 复制偏移量 */
	char sender[CLUSTER_NAMELEN]; /* 发送节点的nodeId */
	unsigned char myslots[CLUSTER_SLOTS/8]; /* 发送节点负责的槽信息 */
	char slaveof[CLUSTER_NAMELEN]; /* 如果发送节点是从节点，记录对应主节点的nodeId */
	uint16_t port; /* 端口号 */
	uint16_t flags; /* 发送节点标识,区分主从角色，是否下线等 */
	unsigned char state; /* 发送节点所处的集群状态 */
	unsigned char mflags[3]; /* 消息标识 */
	union clusterMsgData data /* 消息正文 */;
} clusterMsg;
```

##### 消息体
消息体在Redis内部采用clusterMsgData结构声明。消息体clusterMsgData定义发送消息的数据，其中ping、meet、pong都采用cluster MsgDataGossip数组作为消息体数据，实际消息类型使用消息头的type属性区分.
```
union clusterMsgData {
	/* ping,meet,pong消息体*/
	struct {
	/* gossip消息结构数组 */
	clusterMsgDataGossip gossip[1];
	589
	} ping;
	/* FAIL 消息体 */
	struct {
	clusterMsgDataFail about;
	} fail;
	// ...
};
typedef struct {
	char nodename[CLUSTER_NAMELEN]; /* 节点的nodeId */
	uint32_t ping_sent; /* 最后一次向该节点发送ping消息时间 */
	uint32_t pong_received; /* 最后一次接收该节点pong消息时间 */
	char ip[NET_IP_STR_LEN]; /* IP */
	uint16_t port; /* port*/
	uint16_t flags; /* 该节点标识, */
} clusterMsgDataGossip;
```
## 请求路由
### 请求重定向
1. 每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系；
2. 客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16383取余，计算自己的槽和对应节点；
3. 如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端；
4. 如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常；
5. 客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息；
6. 客户端向目标节点发送命令，获取命令执行结果；
![](https://www.showdoc.cc/server/api/common/visitfile/sign/fbe9bebfd158a6806a0c1561cee79872?showdoc=.jpg)
节点对于不属于它的键命令只回复重定向响应，并不负责转发。

槽命中：直接返回。
```
127.0.0.1:6379> cluster keyslot key:test:1
(integer) 866
```
槽不命中：moved异常,回复`MOVED{slot}{ip}{port}`格式重定向信息。
```
127.0.0.1:6379> set key:test:2 value-2
(error) MOVED 9252 127.0.0.1:6380
```
### ASK重定向
Redis集群支持在线迁移槽（slot）和数据来完成水平伸缩，当slot对应的数据从源节点到目标节点迁移过程中,当客户端向正确的节点发送命令时，槽及槽中数据已经被迁移到别的节点了，就会返回ask，这就是ask重定向机制。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/86df9797a0ef98f703c3f8c6fa256512?showdoc=.jpg)

![](https://www.showdoc.cc/server/api/common/visitfile/sign/e188d8ce0ed4c0c90092def6ad5a19ba?showdoc=.jpg)
1. 客户端向目标节点发送命令，目标节点中的槽已经迁移支别的节点上了，此时目标节点会返回ask转向给客户端；
2. 客户端向新的节点发送Asking命令给新的节点，然后再次向新节点发送命令；
3. 新节点执行命令，把命令执行结果返回给客户端；

moved异常与ask异常的异同：
- 两者都是客户端重定向；
- moved异常：槽已经确定迁移，即槽已经不在当前节点；
- ask异常：槽还在迁移中；

### Smart客户端
Smart客户端通过在内部维护slot→node的映射关系，本地就可实现键到节点的查找，从而保证IO效率的最大化，而MOVED重定向负责协助Smart客户端更新slot→node映射。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/d32e495af0ec244f869254d95fdf6417?showdoc=.jpg)
- 每个JedisPool中缓存了slot和节点node的关系；
- key和slot的关系：对key进行CRC16规则进行hash后与16383取余得到的结果就是槽；
- JedisCluster启动时，已经知道key,slot和node之间的关系，可以找到目标节点；
- JedisCluster对目标节点发送命令，目标节点直接响应给JedisCluster；
- 如果JedisCluster与目标节点连接出错，则JedisCluster会知道连接的节点是一个错误的节点；
- 此时JedisCluster会随机节点发送命令，随机节点返回moved异常给JedisCluster；
- JedisCluster会重新初始化slot与node节点的缓存关系，然后向新的目标节点发送命令，目标命令执行命令并向JedisCluster响应；
- 如果命令发送次数超过5次，则抛出异常"Too many cluster redirection!"；

## 故障转移
### 故障发现
Redis Cluster通过ping/pong消息实现故障发现：不需要sentinel。ping/pong不仅能传递节点与槽的对应消息，也能传递其他状态，比如：节点主从状态，节点故障等。故障发现也是通过消息传播机制实现的，主要环节包括：主观下线（pfail）和客观下线（fail）。
### 主观下线
主观下线指某个节点认为另一个节点不可用，即下线状态，这个状态并不是最终的故障判定，只能代表一个节点的意见，可能存在误判情况。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/6f5cf3dbf5ec2e8ce7235192e3d1eb78?showdoc=.jpg)
1. 节点a发送ping消息给节点b，如果通信正常将接收到pong消息，节
点a更新最近一次与节点b的通信时间。
2. 如果节点a与节点b通信出现问题则断开连接，下次会进行重连。如
果一直通信失败，则节点a记录的与节点b最后通信时间将无法更新。
3. 节点a内的定时任务检测到与节点b最后通信时间超高cluster-nodetimeout
时，更新本地对节点b的状态为主观下线（pfail）。

主观下线简单来讲就是，当cluster-note-timeout时间内某节点无法与另一
个节点顺利完成ping消息通信时，则将该节点标记为主观下线状态。

### 客观下线
当某个节点判断另一个节点主观下线后，相应的节点状态会跟随消息在集群内传播。通过Gossip消息传播，集群内节点不断收集到故障节点的下线报告。当半数以上持有槽的**主节点**都标记某个节点是主观下线时。触发客观下线流程。
1. 当消息体内含有其他节点的pfail状态会判断发送节点的状态，如果发送节点是**主节点则对报告的pfail状态处理**，从节点则忽略。
2. 找到pfail对应的节点结构，**更新clusterNode内部下线报告链表**。
3. 根据更新后的下线报告链表告**尝试进行客观下线**。
>下线报告的有效期限是server.cluster_node_timeout*2，主要是针对故障误报的情况。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/e4db6e708ec7f8d203fe6b1b79719de1?showdoc=.jpg)

1. 首先统计有效的下线报告数量，如果小于集群内持有槽的主节点总数的一半则退出。
2. 当下线报告大于槽主节点数量一半时，标记对应故障节点为客观下线状态。
3. 向集群广播一条fail消息，通知所有的节点将故障节点标记为客观下线，fail消息的消息体只包含故障节点的ID。

广播fail消息是客观下线的最后一步，它承担着非常重要的职责：
- 通知集群内所有的节点标记故障节点为客观下线状态并立刻生效。
- 通知故障节点的从节点触发故障转移流程。

### 故障恢复
故障恢复流程分为：
1. 资格检查；
2. 准备选举时间；
3. 发起选举；
4. 选举投票；
5. 替换主节点；
#### 资格检查
每个从节点都要检查最后与主节点断线时间，判断是否有资格替换故障的主节点。如果从节点与主节点断线时间超过`cluster-node-time*cluster-slave-validity-factor`，则当前从节点不具备故障转移资格。参数`cluster-slavevalidity-factor`用于从节点的有效因子，默认为10。
#### 准备选举时间
复制偏移量越大说明从节点延迟越低，那么它应该具有更高的优先级来替换故障主节点。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/84a8b0d33b4cd2812ba6b690340c121b?showdoc=.jpg)
#### 发起选举
发起选举流程：
1. 更新配置纪元：
	- 标示集群内每个主节点的不同版本和当前集群最大的版本
	- 每次集群发生重要事件时，这里的重要事件指出现新的主节点（新加入的或者由从节点转换而来），从节点竞争选举。都会递增集群全局的配置纪元并赋值给相关主节点，用于记录这一关键事件
	- 主节点具有更大的配置纪元代表了更新的集群状态，因此当节点间进行ping/pong消息交换时，如出现slots等关键信息不一致时，以配置纪元更大的一方为准，防止过时的消息状态污染集群。
	配置纪元的应用场景有：
	- 新节点加入。
	- 槽节点映射冲突检测。
	- 从节点投票选举冲突检测。
2. 广播选举消息：在集群内广播选举消息`FAILOVER_AUTH_REQUEST`，并记录已发送过消息的状态，保证该从节点在一个配置纪元内只能发起一次选举。消息内容如同ping消息只是将type类型变为`FAILOVER_AUTH_REQUEST`。

#### 选举投票
只有持有槽的主节点才会处理故障选举消息`FAILOVER_AUTH_REQUEST`，因为每个持有槽的节点在一个配置纪元
内都有唯一的一张选票，当接到第一个请求投票的从节点消息时回复`FAILOVER_AUTH_ACK`消息作为投票，之后相同配置纪元内其他从节点的选举消息将忽略。
![](https://www.showdoc.cc/server/api/common/visitfile/sign/45cfd8d55a0f2fc01d629b42bfdf0409?showdoc=.jpg)
>投票作废：每个配置纪元代表了一次选举周期，如果在开始投票之后的cluster-node-timeout*2时间内从节点没有获取足够数量的投票，则本次选举作废。从节点对配置纪元自增并发起下一轮投票，直到选举成功为止。

#### 替换主节点
当从节点收集到足够的选票之后，触发替换主节点操作：
1. 当前从节点取消复制变为主节`slaveof no one`；
2. 执行`clusterDelSlot`操作撤销故障主节点负责的槽，并执行`clusterAddSlot`把这些槽委派给自己；
3. 向集群广播自己的pong消息，通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息；

### 故障转移时间
估算出故障转移时间：
1. 主观下线（pfail）识别时间=cluster-node-timeout；
2. 主观下线状态消息传播时间<=cluster-node-timeout/2；
3. 从节点转移时间<=1000毫秒。由于存在延迟发起选举机制，偏移量最大的从节点会最多延迟1秒发起选举；
cluster-node-timeout参数默认15秒。
>failover-time(毫秒) ≤ cluster-node-timeout + cluster-node-timeout/2 + 1000

## 集群运维
### 集群完整性
为了保证集群完整性，默认情况下当集群16384个槽任何一个没有指派到节点时整个集群不可用。执行任何键命令返回（error）CLUSTERDOWN Hash slot not served错误。建议将参数cluster-require-full-coverage配置为no。
### 带宽消耗
集群内所有节点通过ping/pong消息彼此交换信息，节点间消息通信对带宽的消耗体现在以下几个方面：
- 消息发送频率：跟cluster-node-timeout密切相关，当节点发现与其他节点最后通信时间超过cluster-node-timeout/2时会直接发送ping消息；
- 消息数据量：每个消息主要的数据占用包含：slots槽数组（2KB空间）和整个集群1/10的状态数据（10个节点状态数据约1KB）；
- 节点部署的机器规模：机器带宽的上线是固定的，因此相同规模的集群分布的机器越多每台机器划分的节点越均匀，则集群内整体的可用带宽越高；

优化策略：
- 在满足业务需要的情况下尽量避免大集群；
- 适度设置cluster-node-timeout：带宽和故障转移速度的均衡；
- 如果条件允许集群尽量均匀部署在更多机器上；

### Pub/Sub广播问题
在集群模式下内部实现对所有的publish命令都会向所有的节点进行广播，造成每条publish数据都会在集群内所有节点传播一次，加重带宽负担。
解决办法：需要使用Pub/Sub时，为了保证高可用，可以单独开启一套Redis Sentinel。
### 集群倾斜
#### 数据倾斜
数据倾斜主要分为以下几种：
- 节点和槽分配严重不均。
- 不同槽对应键数量差异过大。
- 集合对象包含大量元素。
- 内存相关配置不一致

- 节点和槽分配严重不均
- 不同槽对应键数量差异过大
- 集合对象包含大量元素
- 内存相关配置不一致

#### 请求倾斜
避免方式如下：
- 合理设计键，热点大集合对象做拆分或使用hmget替代hgetall避免整体读取；
- 不要使用热键作为hash_tag，避免映射到同一槽；
- 对于一致性要求不高的场景，客户端可使用本地缓存减少热键调用；
## 总结
- Redis集群数据分区规则采用虚拟槽方式，所有的键映射到16384个槽中，每个节点负责一部分槽和相关数据，实现数据和请求的负载均衡；
- 搭建集群划分三个步骤：准备节点，节点握手，分配槽。可以使用redis-trib.rb create命令快速搭建集群；
- 集群内部节点通信采用Gossip协议彼此发送消息，消息类型分为：ping消息、pong消息、meet消息、fail消息等；
- 集群伸缩通过在节点之间移动槽和相关数据实现：
	- 扩容时根据槽迁移计划把槽从源节点迁移到目标节点；
	- 收缩时如果下线的节点有负责的槽需要迁移到其他节点，再通过cluster forget命令让集群内其他节点忘记被下线节点；
- 使用Smart客户端操作集群达到通信效率最大化，客户端内部负责计算维护键→槽→节点的映射，用于快速定位键命令到目标节点；
- 集群自动故障转移过程分为故障发现和故障恢复。节点下线分为主观下线和客观下线，当超过半数主节点认为故障节点为主观下线时标记它为客观下线状态。从节点负责对客观下线的主节点触发故障恢复流程，保证集群的可用性；
