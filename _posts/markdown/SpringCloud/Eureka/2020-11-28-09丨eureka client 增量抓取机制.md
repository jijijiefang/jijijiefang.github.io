---
layout:     post
title:      "Eureka-09丨client增量抓取机制"
date:       2020-11-28 21:57:44
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# eureka client增量抓取机制
![eureka client 增量抓取注册表机](https://s3.ax1x.com/2020/11/28/D6Gux1.png)
## 每隔30S抓取注册表
```java
    private void initScheduledTasks() {
        //初始化抓取注册表定时任务
        if (clientConfig.shouldFetchRegistry()) {
            // registry cache refresh timer
            //抓取注册表间隔-秒
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            //重试最大次数
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            //创建并执行延时任务-每隔30S抓取注册表
            scheduler.schedule(
                    new TimedSupervisorTask(
                            "cacheRefresh",
                            scheduler,
                            cacheRefreshExecutor,
                            registryFetchIntervalSeconds,
                            TimeUnit.SECONDS,
                            expBackOffBound,
                            new CacheRefreshThread()
                    ), 
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
        ...
```
## 增量抓取策略
```java
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

        try {
            // If the delta is disabled or if it is the first time, get all
            // applications
            Applications applications = getApplications();

            if (...)
            {
                ...
                //全量抓取策略
                getAndStoreFullRegistry();
            } else {
                //增量抓取策略
                getAndUpdateDelta(applications);
            }
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } catch (Throwable e) {
            logger.error(PREFIX + appPathIdentifier + " - was unable to refresh its cache! status = " + e.getMessage(), e);
            return false;
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }

        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();

        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();

        // registry was fetched successfully, so return true
        return true;
    }
```
## 调用restful接口
使用`EurekaHttpClient.getDelta()`，GET请求http://localhost:8080/v2/apps/delta。
## eureka server端查询增量数据
`ApplicationsResource`中`getContainerDifferential`()方法为获取增量数据。使用`ALL_APPS_DELTA`从`readOnlyCacheMap`中获取数据，而`ALL_APPS_DELTA`的数据放入`readWriteCacheMap`时做区分。

```java
    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS_DELTA,
            keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
    );
```
readWriteCacheMap加载数据
```java
private Value generatePayload(Key key) {
        Stopwatch tracer = null;
        try {
            String payload;
            switch (key.getEntityType()) {
                case Application:
                    boolean isRemoteRegionRequested = key.hasRegions();

                    if (ALL_APPS.equals(key.getName())) {
                        ...
                    } else if (ALL_APPS_DELTA.equals(key.getName())) {
                        if (isRemoteRegionRequested) {
                            tracer = serializeDeltaAppsWithRemoteRegionTimer.start();
                            versionDeltaWithRegions.incrementAndGet();
                            versionDeltaWithRegionsLegacy.incrementAndGet();
                            payload = getPayLoad(key,
                                    registry.getApplicationDeltasFromMultipleRegions(key.getRegions()));
                        } else {
                            tracer = serializeDeltaAppsTimer.start();
                            versionDelta.incrementAndGet();
                            versionDeltaLegacy.incrementAndGet();
                            payload = getPayLoad(key, registry.getApplicationDeltas());
                        }
                    } else {
                        tracer = serializeOneApptimer.start();
                        payload = getPayLoad(key, registry.getApplication(key.getName()));
                    }
                    break;
                    ...
            }
            return new Value(payload);
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }
    }
```
## recentlyChangedQueue增量更新机制
`recentlyChangedQueue`中存放新注册、下线等服务实例。在`Registry`构造时定义一个Timer定时任务，每隔**30S**查看服务实例变更记录，是否在队列中停留超过**180S**，如果超过就删除。这个queue只保留3分钟的服务实例变更记录。

```java
private ConcurrentLinkedQueue<RecentlyChangedItem> recentlyChangedQueue = new ConcurrentLinkedQueue<RecentlyChangedItem>();
//定时器执行每隔30S
this.deltaRetentionTimer.schedule(getDeltaRetentionTask(),
        serverConfig.getDeltaRetentionTimerIntervalInMs(),
        serverConfig.getDeltaRetentionTimerIntervalInMs());
//剔除超过3分钟的服务实例信息
private TimerTask getDeltaRetentionTask() {
    return new TimerTask() {
        @Override
        public void run() {
            Iterator<RecentlyChangedItem> it = recentlyChangedQueue.iterator();
            while (it.hasNext()) {
                if (it.next().getLastUpdateTime() <
                        System.currentTimeMillis() - serverConfig.getRetentionTimeInMSInDeltaQueue()) {
                    it.remove();
                } else {
                    break;
                }
            }
        }

    };
}
```
## 增量更新合并至本地注册表

```java
private void getAndUpdateDelta(Applications applications) throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    Applications delta = null;
    EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        delta = httpResponse.getEntity();
    }

    if (delta == null) {
        ...
        getAndStoreFullRegistry();
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
        String reconcileHashCode = "";
        if (fetchRegistryUpdateLock.tryLock()) {
            try {
                //增量更新合并至本地注册表
                updateDelta(delta);
                reconcileHashCode = getReconcileHashCode(applications);
            } finally {
                fetchRegistryUpdateLock.unlock();
            }
        } else {
            logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
        }
        // There is a diff in number of instances for some reason
        //对比hashcode
        if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
            reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
        }
    } else {
        
    }
}
    
private void updateDelta(Applications delta) {
    int deltaCount = 0;
    for (Application app : delta.getRegisteredApplications()) {
        for (InstanceInfo instance : app.getInstances()) {
            Applications applications = getApplications();
            String instanceRegion = instanceRegionChecker.getInstanceRegion(instance);
            if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                Applications remoteApps = remoteRegionVsApps.get(instanceRegion);
                if (null == remoteApps) {
                    remoteApps = new Applications();
                    remoteRegionVsApps.put(instanceRegion, remoteApps);
                }
                applications = remoteApps;
            }

            ++deltaCount;
            if (ActionType.ADDED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                logger.debug("Added instance {} to the existing apps in region {}", instance.getId(), instanceRegion);
                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);
            } else if (ActionType.MODIFIED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                logger.debug("Modified instance {} to the existing apps ", instance.getId());

                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);

            } else if (ActionType.DELETED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                logger.debug("Deleted instance {} to the existing apps ", instance.getId());
                applications.getRegisteredApplications(instance.getAppName()).removeInstance(instance);
            }
        }
    }
    logger.debug("The total number of instances fetched by the delta processor : {}", deltaCount);

    getApplications().setVersion(delta.getVersion());
    getApplications().shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());

    for (Applications applications : remoteRegionVsApps.values()) {
        applications.setVersion(delta.getVersion());
        applications.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
    }
}
```
## 对比hashcode
增量更新合并至本地注册表后，计算hashcode与eureka server端返回的delta里的appsHashCode进行对比，如果不一样会重新从eureka server拉取全量注册表更新到本地缓存。
```java
// There is a diff in number of instances for some reason
if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
    reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
}
//全量抓取注册表并记录对比异常信息
private void reconcileAndLogDifference(Applications delta, String reconcileHashCode) throws Throwable {
    EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
            ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
            : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
    Applications serverApps = httpResponse.getEntity();
    ...
    if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(serverApps));
        getApplications().setVersion(delta.getVersion());
        ...
    } else {
        ...
    }
}
```
## 亮点
### 增量数据
如果保存一份增量的最新变更数据，可以基于LinkedQuueue，将最新变更的数据放入这个queue中，然后后台使用定时任务，每隔一定时间，将在队列中存放超过一定时间的数据移除，保持队列中就是最近几分钟内的变更的增量数据。
### 数据同步的hash值比对机制
在分布式系统里，进行数据的同步，可以采用Hash值的思想：从一个地方的数据计算一个hash值，到另外一个地方，计算一个hash值，保证两个hash值是一样的，确保这个数据传输过程中，没有出什么问题。