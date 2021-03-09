---
layout:     post
title:      "Eureka-07丨Eureka Server 启动流程"
date:       2019-12-10 15:46:19
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# Eureka Server 启动流程
## 1、启动EurekaServer
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
## 2、@EnableEurekaServer

```java
@EnableDiscoveryClient
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {
}
```
将配置类 `EurekaServerMarkerConfiguration` 加入spring 容器。

## 3、EurekaServerMarkerConfiguration
```java
/**
 * 激活 EurekaServerAutoConfiguration 的开关
 */
@Configuration
public class EurekaServerMarkerConfiguration {
   @Bean
   public Marker eurekaServerMarkerBean() {
      return new Marker();
   }
   class Marker {
   }
}
```
只是实例化了一个空类，没有任何实现，从注释中可以看到 `EurekaServerMarkerConfiguration` 是一个激活 `EurekaServerAutoConfiguration` 的开关，在源码的 META-INF/spring.factories中看到
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```
真正的配置信息在 `EurekaServerAutoConfiguration` 中，`@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)` 只有存在 Marker 实例的时候，才会继续加载配置项，这也就要求必须有 `@EnableEurekaServer`注解，才能正常的启动。

## 4、EurekaServerAutoConfiguration
```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
      InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter
```
1. @Configuration 表明这是一个配置类;
2. `@Import(EurekaServerInitializerConfiguration.class)`导入启动EurekaServer的bean;
3. `@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)` 这个是表示只有在spring容器里面含有`Marker`这个实例的时候，才会加载当前这个Bean（`EurekaServerAutoConfiguration` ），这个就是控制是否开启EurekaServer的关键;
4. `@EnableConfigurationProperties({ EurekaDashboardProperties.class, InstanceRegistryProperties.class })`加载配置;
5. `@PropertySource(“classpath:/eureka/server.properties”)` 加载配置文件;

## 5、EurekaServerAutoConfiguration
`EurekaServerInitializerConfiguration` 实现了`SmartLifecycle.start`方法，在容器所有bean加载和初始化完毕执行。

`EurekaServerInitializerConfiguration.start()`

```java
@Override
public void start() {
   new Thread(new Runnable() {
      @Override
      public void run() {
         try {
            //初始化EurekaServer，同时启动Eureka Server
            eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
            log.info("Started Eureka Server");
            // 发布EurekaServer的注册事件
            publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
            // 设置启动的状态为true
            EurekaServerInitializerConfiguration.this.running = true;
            // 发布Eureka Start 事件，另外还有其他事件，可以根据需要监听这些事件，然后做一些特定的业务需求
            publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
         }catch (Exception ex) {
            // Help!
            log.error("Could not initialize Eureka servlet context", ex);
         }
      }
   }).start();
}
```
## 6、contextInitialized()
`EurekaServerBootstrap.contextInitialized()`

```java
public void contextInitialized(ServletContext context) {
   try {
      //初始化运行环境
      initEurekaEnvironment();
      //初始化上下文
      initEurekaServerContext();
      context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
   }catch (Throwable e) {
      log.error("Cannot bootstrap eureka server :", e);
      throw new RuntimeException("Cannot bootstrap eureka server :", e);
   }
}
```

#### EurekaServerBootstrap.initEurekaEnvironment()
```java
protected void initEurekaEnvironment() throws Exception {
   log.info("Setting the eureka configuration..");
   //设置数据中心
   String dataCenter = ConfigurationManager.getConfigInstance()
         .getString(EUREKA_DATACENTER);
   if (dataCenter == null) {
      log.info(
            "Eureka data center value eureka.datacenter is not set, defaulting to default");
      ConfigurationManager.getConfigInstance()
            .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
   }else {
      ConfigurationManager.getConfigInstance()
            .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
   }
   //设置上下文
   String environment = ConfigurationManager.getConfigInstance()
         .getString(EUREKA_ENVIRONMENT);
   if (environment == null) {
      ConfigurationManager.getConfigInstance()
            .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
      log.info(
            "Eureka environment value eureka.environment is not set, defaulting to test");
   }else {
      ConfigurationManager.getConfigInstance()
            .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
   }
}
```
#### EurekaServerBootstrap.initEurekaServerContext()
```java
protected void initEurekaServerContext() throws Exception {
   //向后兼容
   JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
         XStream.PRIORITY_VERY_HIGH);
   XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
         XStream.PRIORITY_VERY_HIGH);
   if (isAws(this.applicationInfoManager.getInfo())) {
      this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
            this.eurekaClientConfig, this.registry, this.applicationInfoManager);
      this.awsBinder.start();
   }
   EurekaServerContextHolder.initialize(this.serverContext);
   log.info("Initialized server context");
   //从邻近的eureka节点复制注册表
   int registryCount = this.registry.syncUp();
   this.registry.openForTraffic(this.applicationInfoManager, registryCount);
   //注册所有监控统计信息
   EurekaMonitors.registerAllStats();
}
```
#### PeerAwareInstanceRegistry.syncUp()
```java
@Override
public int syncUp() {
    int count = 0;
    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                //每次休眠30S
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
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
#### AbstractInstanceRegistry.register()
注册新实例
```java
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            //从registry中获取实例Map
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                //如果获取的为null，新建一个放入registry
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            //存在实例信息
            if (existingLease != null && (existingLease.getHolder() != null)) {
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                //比较实例最后修改时间
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    registrant = existingLease.getHolder();
                }
            } else {
                //加锁
                synchronized (lock) {
                    //每分钟续约次数
                    if (this.expectedNumberOfRenewsPerMin > 0) {
                        this.expectedNumberOfRenewsPerMin = this.expectedNumberOfRenewsPerMin + 2;
                        //15分钟内续约服务的比例小于0.85，则开启自我保护机制，再此期间不会清除已注册的任何服务(默认0.85)
                        this.numberOfRenewsPerMinThreshold =
                                (int) (this.expectedNumberOfRenewsPerMin * serverConfig.getRenewalPercentThreshold());
                    }
                }
            }
            //leaseDuration持续时间默认90s
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                //设置注册时间
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            //最近注册队列加锁
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            //实例信息覆盖状态变化
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);
            
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            //设置实例操作类型
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        } finally {
            read.unlock();
        }
    }
```