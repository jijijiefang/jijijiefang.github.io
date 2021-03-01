---
layout:     post
title:      "SpringCloud-01丨SpringCloud时效性和超时重试机制"
date:       2021-01-06 23:03:42
author:     "jiefang"
header-style: text
tags:
    - SpringCloud
---
# SpringCloud时效性和超时重试机制

## 注册中心

### 服务注册

`SpringCloud`封装`EurekaClient`,服务启动就去注册中心注册，时效性基本在**毫秒级**。

```java
public abstract class AbstractAutoServiceRegistration<R extends Registration>
      implements AutoServiceRegistration, ApplicationContextAware,ApplicationListener<WebServerInitializedEvent> {
    @Override
	@SuppressWarnings("deprecation")
	public void onApplicationEvent(WebServerInitializedEvent event) {
		bind(event);
	}     
	@Deprecated
	public void bind(WebServerInitializedEvent event) {
		ApplicationContext context = event.getApplicationContext();
		if (context instanceof ConfigurableWebServerApplicationContext) {
			if ("management".equals(((ConfigurableWebServerApplicationContext) context)
					.getServerNamespace())) {
				return;
			}
		}
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}
	public void start() {
		if (!isEnabled()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Discovery Lifecycle disabled. Not starting");
			}
			return;
		}
		//没有运行过就注册eureka
		if (!this.running.get()) {
			this.context.publishEvent(
					new InstancePreRegisteredEvent(this, getRegistration()));
			register();
			if (shouldRegisterManagement()) {
				registerManagement();
			}
			this.context.publishEvent(
					new InstanceRegisteredEvent<>(this, getConfiguration()));
			this.running.compareAndSet(false, true);
		}
	}
	protected void register() {
		this.serviceRegistry.register(getRegistration());
	}    
}
```

#### EurekaServiceRegistry#register()

```java
@Override
public void register(EurekaRegistration reg) {
   maybeInitializeClient(reg);

   if (log.isInfoEnabled()) {
      log.info("Registering application "
            + reg.getApplicationInfoManager().getInfo().getAppName()
            + " with eureka with status "
            + reg.getInstanceConfig().getInitialStatus());
   }

   reg.getApplicationInfoManager()
         .setInstanceStatus(reg.getInstanceConfig().getInitialStatus());

   reg.getHealthCheckHandler().ifAvailable(healthCheckHandler -> reg
          //注册EurekaClient                                 
         .getEurekaClient().registerHealthCheck(healthCheckHandler));
}
```

### 服务发现

- 服务启动时发现其它所有实例信息，抓取全量注册表，时效性**毫秒级**；

- 服务新增实例，其它服务对它进行服务发现，时效性**分钟级**；

#### 多级缓存同步

服务实例注册到Eureka的注册表(`registry`)，**readWriteCacheMap**会失效该服务的所有实例；每隔**responseCacheUpdateIntervalMs**（默认**30S**），`readWriteCacheMap`中实例信息同步至`readOnlyCacheMap`中；

```java
public class ResponseCacheImpl implements ResponseCache {
    ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, AbstractInstanceRegistry registry) {
        if (shouldUseReadOnlyResponseCache) {
            timer.schedule(getCacheUpdateTask(),
                    new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                            + responseCacheUpdateIntervalMs),
                    responseCacheUpdateIntervalMs);    
        }
    }
    private TimerTask getCacheUpdateTask() {
        return new TimerTask() {
            @Override
            public void run() {
                logger.debug("Updating the client cache from response cache");
                for (Key key : readOnlyCacheMap.keySet()) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Updating the client cache from response cache for key : {} {} {} {}",
                                     key.getEntityType(), key.getName(), key.getVersion(), key.getType());
                    }
                    try {
                        CurrentRequestVersion.set(key.getVersion());
                        Value cacheValue = readWriteCacheMap.get(key);
                        Value currentCacheValue = readOnlyCacheMap.get(key);
                        if (cacheValue != currentCacheValue) {
                            readOnlyCacheMap.put(key, cacheValue);
                        }
                    } catch (Throwable th) {
                        logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), th);
                    } finally {
                        CurrentRequestVersion.remove();
                    }
                }
            }
        };
    }
}
```

#### EurekaClient增量拉取

EurekaClient构造时初始化周期执行定时任务刷新本地注册表，每隔**30S**执行一次，**registryFetchIntervalSeconds**默认30S。

```java
public class DiscoveryClient implements EurekaClient {
    private void initScheduledTasks() {
         if (clientConfig.shouldFetchRegistry()) {
            //默认30S刷新EurekaClient本地注册表
            int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
            int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
            cacheRefreshTask = new TimedSupervisorTask(
                    "cacheRefresh",
                    scheduler,
                    cacheRefreshExecutor,
                    registryFetchIntervalSeconds,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new CacheRefreshThread()
            );
            scheduler.schedule(
                    cacheRefreshTask,
                    registryFetchIntervalSeconds, TimeUnit.SECONDS);
        }
    }
}
```

### 服务心跳

EurekaClient构造时初始化周期执行定时任务发送心跳，每隔**30S**执行一次，**renewalIntervalInSecs**默认30S。

```java
public class DiscoveryClient implements EurekaClient {
    private void initScheduledTasks() {
        if (clientConfig.shouldRegisterWithEureka()) {
            int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
            int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
            logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

            // Heartbeat timer
            heartbeatTask = new TimedSupervisorTask(
                    "heartbeat",
                    scheduler,
                    heartbeatExecutor,
                    renewalIntervalInSecs,
                    TimeUnit.SECONDS,
                    expBackOffBound,
                    new HeartbeatThread()
            );
            scheduler.schedule(
                    heartbeatTask,
                    renewalIntervalInSecs, TimeUnit.SECONDS); 		   
    }
}    
```

### 服务故障自动感知

- `EurekaServer`初始化上下文时，注册一个`EvictionTask`，每隔60S执行一次，剔除过期实例；`eureka.server.evictionIntervalTimerInMs=60 * 1000`

- 服务实例过期判断机制：90 S（默认）* 2没有心跳，才会认为过期；
- 服务实例故障，其它实例感知，**极端情况时效性为5分钟**；

```java
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
    protected void postInit() {
        renewsLastMin.start();
        if (evictionTaskRef.get() != null) {
            evictionTaskRef.get().cancel();
        }
        //每隔1分钟执行一次剔除过期实例的任务
        evictionTaskRef.set(new EvictionTask());
        evictionTimer.schedule(evictionTaskRef.get(),
                serverConfig.getEvictionIntervalTimerInMs(),
                serverConfig.getEvictionIntervalTimerInMs());
    }
}
```

#### AbstractInstanceRegistry#evict()

```java
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");
	//自我保护机制
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }
	
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                //判断实例是否过期
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }

    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;

    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    if (toEvict > 0) {
        logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

        Random random = new Random(System.currentTimeMillis());
        for (int i = 0; i < toEvict; i++) {
            // Pick a random item (Knuth shuffle algorithm)
            int next = i + random.nextInt(expiredLeases.size() - i);
            Collections.swap(expiredLeases, i, next);
            Lease<InstanceInfo> lease = expiredLeases.get(i);

            String appName = lease.getHolder().getAppName();
            String id = lease.getHolder().getId();
            EXPIRED.increment();
            logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
            internalCancel(appName, id, false);
        }
    }
}
```

#### Lease#isExpired()

```java
public class Lease<T> {
    public static final int DEFAULT_DURATION_IN_SECS = 90;
    public boolean isExpired(long additionalLeaseMs) {
        return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
    }
}
```

#### PeerAwareInstanceRegistryImpl#isLeaseExpirationEnabled()

```java
@Override
public boolean isLeaseExpirationEnabled() {
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```

### 服务下线感知

如果服务正常下线，执行`DiscoveryClient`的`shutown()`方法，此时其他服务感知到这个服务实例下线，也是在**1分钟以内**。

```java
public class DiscoveryClient implements EurekaClient {
    @PreDestroy
    @Override
    public synchronized void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            logger.info("Shutting down DiscoveryClient ...");

            if (statusChangeListener != null && applicationInfoManager != null) {
                applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
            }

            cancelScheduledTasks();

            // If APPINFO was registered
            if (applicationInfoManager != null
                    && clientConfig.shouldRegisterWithEureka()
                    && clientConfig.shouldUnregisterOnShutdown()) {
                applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
                unregister();
            }

            if (eurekaTransport != null) {
                eurekaTransport.shutdown();
            }

            heartbeatStalenessMonitor.shutdown();
            registryStalenessMonitor.shutdown();

            Monitors.unregisterObject(this);

            logger.info("Completed shut down of DiscoveryClient");
        }
    }
}
```

### 自我保护机制

直接关闭自我保护机制，`eureka.server.enableSelfPreservation = false`

### EurekaServer集群负载均衡

`EurekaClient`默认找的是`application.yml`中的`EurekaServer`集群配置的第一个实例作为注册中心，如果这个`EurekaServer`宕机，重试后会访问其它配置的`EurekaServer`。

### EurekaServer集群同步

1. 集群数据同步的任务，放入一个`acceptorQueue`中；
2. `AcceptorRunner`后台线程每隔10ms执行一次业务逻辑；
3. 默认将500ms内的请求，注册、心跳、下线，打成一个batch，再一次性将这个batch发送给其他的机器，减少网络通信的次数，减少网络通信的开销；
4. **eureka server集群同步的时效性，基本上是在1s内，几百毫秒都是正常的**；

## 服务调用

###  ribbon + eureka服务发现与故障感知的时效性

ribbon的`PollingServerListUpdater`周期执行任务，每隔30S更新EurekaClient的注册表到ribbon本地。

- ribbon感知新增服务实例，1分钟到1.5分钟；
- ribbon感知服务实例宕机，4分钟到5.5分钟；

```java
public class PollingServerListUpdater implements ServerListUpdater {
    @Override
    public synchronized void start(final UpdateAction updateAction) {
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
                        updateAction.doUpdate();
                        lastUpdated = System.currentTimeMillis();
                    } catch (Exception e) {
                        logger.warn("Failed one update cycle", e);
                    }
                }
            };

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
}
```

###  ribbon负载均衡算法

ribbon默认负载均衡器为**ZoneAwareLoadBalancer**，优先访问同机房，然后同机房内使用round robin轮训算法做负载均衡。

### feign+ribbon超时与重试

hystrix是包裹了ribbno的使用的，一般来说，hystrix的超时时长必须大于ribbon的超时时长，必须是hystrix超时时长最好是远大于ribbon超时时长，超时和重试尽量都是以ribbon为主。hystrix的超时时间计算公式如下：

> `(ribbon.ConnectTimeout + ribbon.ReadTimeout) * (ribbon.MaxAutoRetries + 1) * (ribbon.MaxAutoRetriesNextServer + 1)`

```yaml
ribbon:
	ConnectTimeout: 1000
	ReadTimeout: 1000
	OkToRetryOnAllOperations: true
 	MaxAutoRetries: 1
 	MaxAutoRetriesNextServer: 1
```

- `ConnectTimeout`：连接一台机器的超时时间；
- `ReadTimeout`：向一台机器发起请求的时间；
- `OkToRetryOnAllOperations`：对所有的操作请求包括get、post、put都进行重试；

#### 示例

```yaml
ribbon:
  ConnectTimeout: 1000
  ReadTimeout: 1000
  OkToRetryOnAllOperations: true
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 3
```

请求8088异常，进行重试

第一次重试，还是8088；

第二次重试，还是8088；

第三次重试，还是8088；

第四次重试，才是8080；

```yaml
ribbon:
  ConnectTimeout: 1000
  ReadTimeout: 1000
  OkToRetryOnAllOperations: true
  MaxAutoRetries: 1
  MaxAutoRetriesNextServer: 1
```

请求8088异常，进行重试

第一次重试，还是8088；

第二次重试，就是8080；

## 服务网关

### ribbon预加载

Ribbon进行客户端负载均衡的Client并不是在服务启动的时候就初始化好的，而是在调用的时候才会去创建相应的Client，所以第一次调用的耗时不仅仅包含发送HTTP请求的时间，还包含了创建RibbonClient的时间，这样如果创建时间速度较慢，同时设置的超时时间又比较短的话,容易超时。

```yaml
ribbon:
	eager-load:
  		enabled: true
```

### 网关超时重试

如果不配置ribbon的超时时间，默认的hystrix超时时间是4000ms（4s）。

不设置网关重试，使用`RibbonLoadBalancingHttpClient`；设置网关重试后，使用`RetryableRibbonLoadBalancingHttpClient`。

```xml
<dependency>
	<groupId>org.springframework.retry</groupId>
	<artifactId>spring-retry</artifactId>
</dependency>
```

```yaml
zuul:
  retryable: true
ribbon:
  ReadTimeout:1000
  ConnectTimeout:1000
  MaxAutoRetries:1
  MaxAutoRetriesNextServer:1
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000  
```

