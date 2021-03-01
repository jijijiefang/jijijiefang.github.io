---
layout:     post
title:      "Eureka-13丨eureka server自我保护机制"
date:       2020-12-01 00:01:38
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# eureka server自我保护机制
Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 **85%**，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期。

使用参数`eureka.server.enable-self-preservation=false`来开启或者关闭自我保护机制。
```
//Eureka Server 不能少于的收到客户端实例续约数量。
protected volatile int numberOfRenewsPerMinThreshold = 实例数量 * 2 * 85%;
//Eureka Server 期望每分钟收到客户端实例续约的总数。
protected volatile int expectedNumberOfRenewsPerMin = 实例数量 * 2;
```
## EvictionTask
eureka server移除失效服务实例的定时任务。
```
//registry注册表初始化时，提交周期延时任务，每分钟执行一次
protected void postInit() {
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    evictionTaskRef.set(new EvictionTask());
    evictionTimer.schedule(evictionTaskRef.get(),
            serverConfig.getEvictionIntervalTimerInMs(),
            serverConfig.getEvictionIntervalTimerInMs());
}
class EvictionTask extends TimerTask {
    ...
    @Override
    public void run() {
        try {
            long compensationTimeMs = getCompensationTimeMs();
            logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
            evict(compensationTimeMs);
        } catch (Throwable e) {
            logger.error("Could not run the evict task", e);
        }
    }
    ...
}
```
### 移除失效实例方法
在移除失效实例时会根据是否开启自我保护机制和期望心跳次数与过去一分钟内收到心跳次数比较，来决定是否进行移除实例。
```
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }
    //遍历获得失效的服务实例放入list
    //随机移除失效的服务实例    
}
```
### PeerAwareInstanceRegistryImpl.isLeaseExpirationEnabled()

```
@Override
public boolean isLeaseExpirationEnabled() {
    //自我保护机制没开启，直接返回可以进行失效服务实例移除
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    //开启的话，需要判断判断上一分钟的心跳次数是否小于期望的一分钟心跳次数
    //如果小于，则不进行服务实例移除操作
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```

## 期望心跳次数初始化
eureka初始化时会根据服务实例数量计算出每分钟期望心跳次数。
`EurekaBootStrap#initEurekaServerContext()->PeerAwareInstanceRegistryImpl.openForTraffic(ApplicationInfoManager applicationInfoManager, int count)`

```
    @Override
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
        //eureka版本1.7直接硬编码
        //期望每分钟心跳次数=实例数量*2
        this.expectedNumberOfRenewsPerMin = count * 2;
        //每分钟心跳次数开启自我保护机制的阈值=期望每分钟心跳*85%
        this.numberOfRenewsPerMinThreshold =
                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
        ...
        super.postInit();
    }
```
## 期望心跳次数修改定时任务
eureka server上下文初始化时，初始化一个定时任务：每15分钟执行一次，根据当前eureka server服务实例数量计算每分钟期望心跳次数。
`EurekaBootStrap#initEurekaServerContext()->DefaultEurekaServerContext#initialize()->PeerAwareInstanceRegistryImpl#init(PeerEurekaNodes peerEurekaNodes)->PeerAwareInstanceRegistryImpl#scheduleRenewalThresholdUpdateTask()->PeerAwareInstanceRegistryImpl#updateRenewalThreshold()`。

```
private void scheduleRenewalThresholdUpdateTask() {
    timer.schedule(new TimerTask() {
                       @Override
                       public void run() {
                           updateRenewalThreshold();
                       }
                   }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
            serverConfig.getRenewalThresholdUpdateIntervalMs());
}

private void  updateRenewalThreshold() {
    try {
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        synchronized (lock) {
            // Update threshold only if the threshold is greater than the
            // current expected threshold of if the self preservation is disabled.
            //如果服务实例数量*2大于期望阈值或自我保护关闭情况下直接更新
            if ((count * 2) > (serverConfig.getRenewalPercentThreshold() * numberOfRenewsPerMinThreshold)
                    || (!this.isSelfPreservationModeEnabled())) {
                this.expectedNumberOfRenewsPerMin = count * 2;
                this.numberOfRenewsPerMinThreshold = (int) ((count * 2) * serverConfig.getRenewalPercentThreshold());
            }
        }
        logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
    } catch (Throwable e) {
        logger.error("Cannot update renewal threshold", e);
    }
}
```
## 期望心跳次数增减
期望心跳次数与服务实例数量相关，随着服务实例上线、下线和故障变化。上线时，期望心跳次数+2，下线时，期望心跳次数-2。故障时移除服务实例没有更新期望心跳次数的代码（BUG），导致期望心跳次数不减少，而实际服务实例变少了，使实际心跳次数减少，所以导致较多服务实例故障被移除的时候，eureka server进入自我保护机制。

## MeasuredRate计数器
上一分钟实际心跳是根据`MeasuredRate`计数器来计算的。
```
//每60秒设置当前计数为0，当前计数赋值为上一分钟计数
this.renewsLastMin = new MeasuredRate(1000 * 60 * 1);
public synchronized void start() {
    if (!isActive) {
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                try {
                    // Zero out the current bucket.
                    lastBucket.set(currentBucket.getAndSet(0));
                } catch (Throwable e) {
                    logger.error("Cannot reset the Measured Rate", e);
                }
            }
        }, sampleInterval, sampleInterval);

        isActive = true;
    }
}
//过去一分钟计数
public long getCount() {
    return lastBucket.get();
}
//当前计数+1
public void increment() {
    currentBucket.incrementAndGet();
}
```
