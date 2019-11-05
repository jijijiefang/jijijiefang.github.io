---
layout:     post
title:      "Zookeeper基础进阶"
date:       2019-11-05 00:00:00
author:     "jiefang"
header-style: text
tags:
    - Zookeeper
---
# Zookeeper基础进阶
## watcher机制
Zk中引入了watcher机制来实现了发布/订阅功能，能够让多个订阅
者同时监听某一个主题对象，当这个主题对象自身状态变化时，会通知所有订阅者ͺ

Watcher组成
- 客户端
- 客户端watchManager 
- Zk服务器

Watcher机制
- 客户端向zk服务器注册watcher的同时，会将watcher对象存储在客户端的watchManager
- Zk服务器触发watcher事件后，会向客户端发送通知，客户端线程从watchManager中݊起watcher执行
![image](https://s2.ax1x.com/2019/09/29/u89MQA.png)

Watcher接口
- public class ZLock implements Watcher
- public void process(WatchedEvent event)

Watcher事件
- 通知状态：org.apache.zookeeper.Watcher.Event.KeeperState
- 事件类型：org.apache.zookeeper.Watcher.Event.EventType

![image](https://s2.ax1x.com/2019/09/29/u8VLLQ.png)

- NodeDataChanged事件
    - 无论节点数据发生变化ଐ是数据版本发生变化都会触发
    - 即使被更新数据跟新数据一样，数据版本都会发生变化（根据dataVersion判断）
- NodeChildrenChanged
    - 新增节点或者删除节点
- AuthFailed
    - 重点不是客户端会话没有权限而是݉权失败

客户端只能收到相关事件通知，但是并不能获取到对应数据节点的原始数据内容以
及变更后的新数据内容ͺ因此，如果业务需要知道变更前的数据或者变更后的新数据，需要业务保存变更前的数据和调用接口获取新的数据

创建zk客户端对象实例时注册
- ZooKeeper(String connectString, int sessionTimeout, Watcher watcher) 
- ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, boolean canBeReadOnly)
- ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd) 
- ZooKeeper(String connectString, int sessionTimeout, Watcher watcher, long sessionId, byte[] sessionPasswd, boolean canBeReadOnly)
通过这种方式注册的watcher将会作为整个zk会话期间的默认watcher，会一直被
保存在客户端ZKWatchManager的defaultWatcher中，如果有其它的设置，则这个
watcher会被覆盖。
其它注册api
- getChildren(String path, Watcher watcher) 
- getChildren(String path, boolean watch) 
    - Boolean watch表示是否使用上下文中默认的watcher，即创建zk实例时设置的watcher 
- getData(String path, boolean watch, Stat stat) 
    - Boolean watch表示是否使用上下文中默认的watcher，即创建zk实例时设置的watcher 
- getData(String path, Watcher watcher, AsyncCallback.DataCallback cb, Object ctx) 
- exists(String path, boolean watch) 
    - Boolean watch表示是否使用上下文中默认的watcher，即创建zk实例时设置的watcher 
- exists(String path, Watcher watcher)
Watcher设置后，一旦触发一次即会失效，如果需要一直监听，就需要再注册
## 客户端watcher注册流程
![image](https://s2.ax1x.com/2019/09/29/u8eqEj.png)
服务器端处理watcher
![image](https://s2.ax1x.com/2019/09/29/u8mpKU.png)
