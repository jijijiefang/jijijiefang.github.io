---
layout:     post
title:      "Ribbon-03丨Ribbon与Eureka整合"
date:       2020-12-09 22:02:38
author:     "jiefang"
header-style: text
tags:
    - Ribbon
    - SpringCloud
---
# ribbon与eureka整合

![ribbon整合eureka](https://s3.ax1x.com/2020/12/09/rCvQ3t.png)

## 拉取注册表

### ZoneAwareLoadBalancer构造方法

`ZoneAwareLoadBalancer`构造初始化，执行父类`DynamicServerListLoadBalancer`构造方法。

```java
public class ZoneAwareLoadBalancer<T extends Server> extends DynamicServerListLoadBalancer<T> {
    public ZoneAwareLoadBalancer(IClientConfig clientConfig, IRule rule,
                                 IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,
                                 ServerListUpdater serverListUpdater) {
        //调用父类构造方法
        super(clientConfig, rule, ping, serverList, filter, serverListUpdater);
    }
}
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
        super(clientConfig, rule, ping);
        this.serverListImpl = serverList;
        this.filter = filter;
        this.serverListUpdater = serverListUpdater;
        if (filter instanceof AbstractServerListFilter) {
            ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
        }
        //拉取注册表
        restOfInit(clientConfig);
    }
    //拉取注册表
    void restOfInit(IClientConfig clientConfig) {
        boolean primeConnection = this.isEnablePrimingConnections();
        // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
        this.setEnablePrimingConnections(false);
        //增量拉取注册表
        enableAndInitLearnNewServersFeature();
        //全量拉取注册表
        updateListOfServers();
        if (primeConnection && this.getPrimeConnections() != null) {
            this.getPrimeConnections()
                    .primeConnections(getReachableServers());
        }
        this.setEnablePrimingConnections(primeConnection);
        LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
    }    
}
```

### 拉取注册表

```java
public class DynamicServerListLoadBalancer<T extends Server> extends BaseLoadBalancer {
    //拉取注册表
    void restOfInit(IClientConfig clientConfig) {
        boolean primeConnection = this.isEnablePrimingConnections();
        // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
        this.setEnablePrimingConnections(false);
        //增量拉取注册表
        enableAndInitLearnNewServersFeature();
        //全量拉取注册表
        updateListOfServers();
        if (primeConnection && this.getPrimeConnections() != null) {
            this.getPrimeConnections()
                    .primeConnections(getReachableServers());
        }
        this.setEnablePrimingConnections(primeConnection);
        LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
    }    
}
```

### 增量拉取注册表

```java
DynamicServerListLoadBalancer#enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
    //serverListUpdater是PollingServerListUpdater，在RibbonClientConfiguration自动装配
    serverListUpdater.start(updateAction);
}
@Override
PollingServerListUpdater#start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                if (!isActive.get()) {
                    if (scheduledFuture != null) {
                        scheduledFuture.cancel(true);
                    }
                    return;
                }
                try {
                    //实现在DynamicServerListLoadBalancer中
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };
        //定时任务，第一次延迟1S执行，以后每隔30S执行
        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
            wrapperRunnable,
            initialDelayMs,
            refreshIntervalMs,
            TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
//DynamicServerListLoadBalancer中doUpdate实现
protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
    @Override
    public void doUpdate() {
        updateListOfServers();
    }
};
```

### updateListOfServers()

增量拉取注册表和全量拉取注册表都调到了这个方法。

```java
@VisibleForTesting
DynamicServerListLoadBalancer#updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    //如果不为空就是增量拉取
    if (serverListImpl != null) {
        //调用子类实现拉取注册表，这里使用的是DiscoveryEnabledNIWSServerList拉取eureka注册表
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                     getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                         getIdentifier(), servers);
        }
    }
    //更新ribbon本地缓存服务注册表
    updateAllServerList(servers);
}
```

### EurekaClient拉取注册表

ribbon使用EurekaClient拉取eureka服务实例注册表。

```java
DiscoveryEnabledNIWSServerList#getUpdatedListOfServers(){
    return obtainServersViaDiscovery();
}
DiscoveryEnabledNIWSServerList#obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

    if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList<DiscoveryEnabledServer>();
    }

    EurekaClient eurekaClient = eurekaClientProvider.get();
    if (vipAddresses!=null){
        for (String vipAddress : vipAddresses.split(",")) {
            // if targetRegion is null, it will be interpreted as the same region of client
            List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
            for (InstanceInfo ii : listOfInstanceInfo) {
                if (ii.getStatus().equals(InstanceStatus.UP)) {

                    if(shouldUseOverridePort){
                        if(logger.isDebugEnabled()){
                            logger.debug("Overriding port on client name: " + clientName + " to " + overridePort);
                        }
                        InstanceInfo copy = new InstanceInfo(ii);
                        if(isSecure){
                            ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                        }else{
                            ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                        }
                    }
                    DiscoveryEnabledServer des = createServer(ii, isSecure, shouldUseIpAddr);
                    serverList.add(des);
                }
            }
            if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
            }
        }
    }
    return serverList;
}
```

### 更新ribbon缓存注册表

```java
DynamicServerListLoadBalancer#updateAllServerList(List<T> ls) {
    // other threads might be doing this - in which case, we pass
    if (serverListUpdateInProgress.compareAndSet(false, true)) {
        try {
            for (T s : ls) {
                //直接设置拉取回来的注册表为存活，不用等ping周期
                s.setAlive(true);
            }
            setServersList(ls);
            super.forceQuickPing();
        } finally {
            serverListUpdateInProgress.set(false);
        }
    }
}
```

