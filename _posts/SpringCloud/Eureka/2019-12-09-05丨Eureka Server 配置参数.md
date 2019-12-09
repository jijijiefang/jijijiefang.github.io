---
layout:     post
title:      "Eureka-05丨Eureka Server 配置参数"
date:       2019-12-09 17:17:46
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# Eureka Server 配置参数（eureka.server.*）
org.springframework.cloud.netflix.eureka.server.EurekaServerConfigBean

## Eureka Server配置

参数名称 | 说明|默认值
---|---|---
eureka.server.enable-self-preservation|启用自我保护机制，默认为true|true
eureka.server.eviction-interval-timer-in-ms|清除无效服务实例的时间间隔（ms），默认1分钟|60000
eureka.server.delta-retention-timer-interval-in-ms|清理无效增量信息的时间间隔（ms），默认30秒|30000
eureka.server.disable-delta|禁用增量获取服务实例信息|false
eureka.server.log-identity-headers|是否记录登录日志|true
eureka.server.rate-limiter-burst-size|限流大小|10
eureka.server.rate-limiter-enabled|是否启用限流|false
eureka.server.rate-limiter-full-fetch-average-rate|平均请求速率|100
eureka.server.rate-limiter-throttle-standard-clients|是否对标准客户端进行限流|false
eureka.server.rate-limiter-registry-fetch-average-rate|服务注册与拉取的平均速率|500
eureka.server.rate-limiter-privileged-clients|信任的客户端列表|
eureka.server.renewal-percent-threshold|15分钟内续约服务的比例小于0.85，则开启自我保护机制，再此期间不会清除已注册的任何服务（即便是无效服务）|0.85
eureka.server.renewal-threshold-update-interval-ms|更新续约阈值的间隔（分钟），默认15分钟|15
eureka.server.response-cache-auto-expiration-in-seconds|注册信息缓存有效时长（s），默认180秒|180
eureka.server.response-cache-update-interval-ms|注册信息缓存更新间隔（s），默认30秒|30
eureka.server.retention-time-in-m-s-in-delta-queue|保留增量信息时长（分钟），默认3分钟|3
eureka.server.sync-when-timestamp-differs|当时间戳不一致时，是否进行同步|true
eureka.server.use-read-only-response-cache|是否使用只读缓存策略|true


## Eureka Server 集群配置


参数名称 | 说明|默认值|
---|---|---
eureka.server.enable-replicated-request-compression|复制数据请求时，数据是否压缩|false
eureka.server.batch-replication|节点之间数据复制是否采用批处理|false
eureka.server.max-elements-in-peer-replication-pool|备份池最大备份事件数量，默认1000|1000
eureka.server.max-elements-in-status-replication-pool|状态备份池最大备份事件数量，默认1000|1000
eureka.server.max-idle-thread-age-in-minutes-for-peer-replication|节点之间信息同步线程最大空闲时间（分钟）|15
eureka.server.max-idle-thread-in-minutes-age-for-status-replication|节点之间状态同步线程最大空闲时间（分钟）|10
eureka.server.max-threads-for-peer-replication|节点之间信息同步最大线程数量|20
eureka.server.max-threads-for-status-replication|节点之间状态同步最大线程数量|1
eureka.server.max-time-for-replication|节点之间信息复制最大通信时长（ms）|30000
eureka.server.min-available-instances-for-peer-replication|集群中服务实例最小数量，-1 表示单节点|-1
eureka.server.min-threads-for-peer-replication|节点之间信息复制最小线程数量|5
eureka.server.min-threads-for-status-replication|节点之间信息状态同步最小线程数量|1
eureka.server.number-of-replication-retries|节点之间数据复制时，可重试次数|5
eureka.server.peer-eureka-nodes-update-interval-ms|节点更新数据间隔时长（分钟）|10
eureka.server.peer-eureka-status-refresh-time-interval-ms|节点之间状态刷新间隔时长（ms）|30000
eureka.server.peer-node-connect-timeout-ms|节点之间连接超时时长（ms）|200
eureka.server.peer-node-connection-idle-timeout-seconds|节点之间连接后，空闲时长（s）|30
eureka.server.peer-node-read-timeout-ms|几点之间数据读取超时时间（ms）|200
eureka.server.peer-node-total-connections|集群中节点连接总数|1000
eureka.server.peer-node-total-connections-per-host|节点之间连接，单机最大连接数量|500
eureka.server.registry-sync-retries|节点启动时，尝试获取注册信息的次数|500
eureka.server.registry-sync-retry-wait-ms|节点启动时，尝试获取注册信息的间隔时长（ms）|30000
eureka.server.wait-time-in-ms-when-sync-empty|在Eureka服务器获取不到集群里对等服务器上的实例时，需要等待的时间（分钟）|5



