---
layout:     post
title:      "Eureka-14丨eureka server集群机制"
date:       2020-12-01 23:28:22
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# eureka server集群机制

![eureka server集群](https://s3.ax1x.com/2020/12/01/DfQqcd.png)

## 定时刷新eureka server集群列表
`EurekaBootStrap`初始化eureka server上下文时，会初始化其它eureka server集群地址，代表其它eureka server的PeerEurekaNode列表peerEurekaNodes。

```
//eureka server上下文初始化
EurekaBootStrap#initEurekaServerContext(){
    ...
    serverContext.initialize();
    ...
}
//默认eureka server上下文初始化
DefaultEurekaServerContext#initialize(){
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}
//eureka server集群节点类启动
PeerEurekaNodes#start() {
    //初始化线程池
    taskExecutor = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                    thread.setDaemon(true);
                    return thread;
                }
            }
    );
    try {
        //先执行一遍更新eureka server集群列表的方法
        updatePeerEurekaNodes(resolvePeerUrls());
        //eureka server集群列表更新Thread
        Runnable peersUpdateTask = new Runnable() {
            @Override
            public void run() {
                try {
                    updatePeerEurekaNodes(resolvePeerUrls());
                } catch (Throwable e) {
                    logger.error("Cannot update the replica Nodes", e);
                }

            }
        };
        //线程池每隔10分钟更新一次eureka server列表
        taskExecutor.scheduleWithFixedDelay(
                peersUpdateTask,
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                TimeUnit.MILLISECONDS
        );
    } catch (Exception e) {
        throw new IllegalStateException(e);
    }
    for (PeerEurekaNode node : peerEurekaNodes) {
        logger.info("Replica node URL:  {}", node.getServiceUrl());
    }
}
//获取其它eureka server节点地址
PeerEurekaNodes#resolvePeerUrls(){
    InstanceInfo myInfo = applicationInfoManager.getInfo();
    String zone = InstanceInfo.getZone(clientConfig.getAvailabilityZones(clientConfig.getRegion()), myInfo);
    //从配置文件获取所有eureka server节点地址
    List<String> replicaUrls = EndpointUtils
            .getDiscoveryServiceUrls(clientConfig, zone, new EndpointUtils.InstanceInfoBasedUrlRandomizer(myInfo));

    int idx = 0;
    while (idx < replicaUrls.size()) {
        //移除自身节点
        if (isThisMyUrl(replicaUrls.get(idx))) {
            replicaUrls.remove(idx);
        } else {
            idx++;
        }
    }
    return replicaUrls;
}
//更新eureka server集群中的节点
PeerEurekaNodes#updatePeerEurekaNodes(List<String> newPeerUrls) {
     if (newPeerUrls.isEmpty()) {
        logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
        return;
    }
    //现有其它eureka server集群列表
    Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
    toShutdown.removeAll(newPeerUrls);
    Set<String> toAdd = new HashSet<>(newPeerUrls);
    toAdd.removeAll(peerEurekaNodeUrls);

    if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
        return;
    }

    // Remove peers no long available
    List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);

    if (!toShutdown.isEmpty()) {
        logger.info("Removing no longer available peer nodes {}", toShutdown);
        int i = 0;
        while (i < newNodeList.size()) {
            PeerEurekaNode eurekaNode = newNodeList.get(i);
            //移除已经要关闭的eureka server节点
            if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                newNodeList.remove(i);
                eurekaNode.shutDown();
            } else {
                i++;
            }
        }
    }

    // Add new peers
    //新增新的eureka server节点
    if (!toAdd.isEmpty()) {
        logger.info("Adding new peer nodes {}", toAdd);
        for (String peerUrl : toAdd) {
            newNodeList.add(createPeerEurekaNode(peerUrl));
        }
    }

    this.peerEurekaNodes = newNodeList;
    this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
}
```
## registry.syncUp()
eureka server自己本身就是一个eureka client，在初始化时会找任意一个eureka server节点拉取全部注册表，获取到所有服务实例信息。

```
//eureka server上下文初始化
EurekaBootStrap#initEurekaServerContext(){
    ...
    // Copy registry from neighboring eureka node
    int registryCount = registry.syncUp();
    ...
}

@Override
PeerAwareInstanceRegistryImpl#syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                //默认每30S同步一次
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        //eureka server作为一个eureka client拉取到的注册表服务实例信息
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        //注册节点，将实例放入recentlyChangedQueue
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```
## 注册、下线、故障、心跳
`ApplicationResource#addInstance()`方法，在自己本地完成注册，`PeerAwareInstanceRegistryImpl#register()->PeerAwareInstanceRegistryImpl#replicateToPeers()`,把这次服务实例注册同步到其它所有eureka server节点，其它eureka server节点接收操作请求只在自己本地执行，不会再次传播到其它eureka server。

```
ApplicationResource#addInstance(InstanceInfo info, @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    ...
    registry.register(info, "true".equals(isReplication));
    ...
}
PeerAwareInstanceRegistryImpl#register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    //调用父类方法，服务实例加入eureka server
    super.register(info, leaseDuration, isReplication);
    //
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
//复制注册、下线、心跳、故障等操作到其它eureka server节点
PeerAwareInstanceRegistryImpl#replicateToPeers(Action action, String appName, String id,InstanceInfo info /* optional */,InstanceStatus newStatus /*optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }
        //循环所有eureka server节点复制，除了自身
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers(Action action, String appName,String id, InstanceInfo info, InstanceStatus newStatus,PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    }
}
//eureka server同步注册操作到其它节点时使用的是replicationClient
PeerEurekaNode#register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            },
            expiryTime
    );
}

AbstractJersey2EurekaHttpClient#register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    Response response = null;
    try {
        Builder resourceBuilder = jerseyClient.target(serviceUrl).path(urlPath).request();
        addExtraProperties(resourceBuilder);
        //此处将isReplication设置为true,这样其它eureka server节点收到注册请求只在本地执行，不会再次同步到其它节点
        addExtraHeaders(resourceBuilder);
       ...
    } finally {
       ...
    }
}
Jersey2ReplicationClient#addExtraHeaders(Builder webResource) {
    webResource.header(PeerEurekaNode.HEADER_REPLICATION, "true");
}
```