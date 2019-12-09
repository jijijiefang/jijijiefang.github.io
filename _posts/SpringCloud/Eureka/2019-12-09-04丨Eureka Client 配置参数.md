---
layout:     post
title:      "Eureka-04丨Eureka Client 配置参数"
date:       2019-12-09 17:16:42
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# Eureka Client 配置参数（eureka.client.*）

org.springframework.cloud.netflix.eureka.EurekaClientConfigBean

参数名称 | 说明|默认值
---|---|---
eureka.client.enabled|用于指示Eureka客户端已启用的标志|true
eureka.client.registry-fetch-interval-seconds|指示从eureka服务器获取注册表信息的频率（s）|30
eureka.client.instance-info-replication-interval-seconds|更新实例信息的变化到Eureka服务端的间隔时间，（s）|30
eureka.client.initial-instance-info-replication-interval-seconds|初始化实例信息到Eureka服务端的间隔时间，（s）|40
eureka.client.eureka-service-url-poll-interval-seconds|询问Eureka Server信息变化的时间间隔（s），默认为300秒	|300
eureka.client.eureka-server-read-timeout-seconds|读取Eureka Server 超时时间（s），默认8秒|8
eureka.client.eureka-server-connect-timeout-seconds|连接Eureka Server 超时时间（s），默认5秒|5
eureka.client.eureka-server-total-connections|获取从eureka客户端到所有eureka服务器的连接总数,默认200个|200
eureka.client.eureka-server-total-connections-per-host|获取从eureka客户端到eureka服务器主机允许的连接总数，默认50个|50
eureka.client.eureka-connection-idle-timeout-seconds|连接到 Eureka Server 空闲连接的超时时间（s），默认30|30
eureka.client.registry-refresh-single-vip-address|指示客户端是否仅对单个VIP的注册表信息感兴趣，默认为null|null
eureka.client.heartbeat-executor-thread-pool-size|心跳保持线程池初始化线程数，默认2个|2
eureka.client.heartbeat-executor-exponential-back-off-bound|心跳超时重试延迟时间的最大乘数值，默认10|10
eureka.client.serviceUrl.defaultZone|可用区域映射到与eureka服务器通信的完全限定URL列表。每个值可以是单个URL或逗号分隔的备用位置列表。(http://${eureka.instance.hostname}:${server.port}/eureka/)||
eureka.client.use-dns-for-fetching-service-urls|指示eureka客户端是否应使用DNS机制来获取要与之通信的eureka服务器列表。当DNS名称更新为具有其他服务器时，eureka客户端轮询eurekaServiceUrlPollIntervalSeconds中指定的信息后立即使用该信息。|false
eureka.client.register-with-eureka|指示此实例是否应将其信息注册到eureka服务器以供其他服务发现，默认为false|True
eureka.client.prefer-same-zone-eureka|实例是否使用同一zone里的eureka服务器，默认为true，理想状态下，eureka客户端与服务端是在同一zone下|true
eureka.client.log-delta-diff|是否记录eureka服务器和客户端之间在注册表的信息方面的差异，默认为false|false
eureka.client.disable-delta|指示eureka客户端是否禁用增量提取|false
eureka.client.fetch-remote-regions-registry|逗号分隔的区域列表，提取eureka注册表信息||
eureka.client.on-demand-update-status-change|客户端的状态更新到远程服务器上，默认为true|true
eureka.client.allow-redirects|指示服务器是否可以将客户端请求重定向到备份服务器/集群。如果设置为false，则服务器将直接处理请求。如果设置为true，则可以将HTTP重定向发送到具有新服务器位置的客户端。|false
eureka.client.availability-zones.*|获取此实例所在区域的可用区域列表（在AWS数据中心中使用）。更改在运行时在registryFetchIntervalSeconds指定的下一个注册表获取周期生效。|
eureka.client.backup-registry-impl|获取实现BackupRegistry的实现的名称，该实现仅在eureka客户端启动时第一次作为后备选项获取注册表信息。 对于需要额外的注册表信息弹性的应用程序，可能需要这样做，否则它将无法运行。||
eureka.client.cache-refresh-executor-exponential-back-off-bound|在发生一系列超时的情况下，它是重试延迟的最大乘数值。|10
eureka.client.cache-refresh-executor-thread-pool-size|缓存刷新线程池初始化线程数量|2
eureka.client.client-data-accept|客户端数据接收的名称|full
eureka.client.decoder-name|解码器名称
eureka.client.dollar-replacement|eureka服务器序列化/反序列化的信息中获取“$”符号的替换字符串。默认为“_-”|
eureka.client.encoder-name|编码器名称||
eureka.client.escape-char-replacement|eureka服务器序列化/反序列化的信息中获取“_”符号的的替换字符串。默认为“__“||
eureka.client.eureka-server-d-n-s-name|获取要查询的DNS名称来获得eureka服务器，此配置只有在eureka服务器ip地址列表是在DNS中才会用到。默认为null|null
eureka.client.eureka-server-port|获取eureka服务器的端口，此配置只有在eureka服务器ip地址列表是在DNS中才会用到。默认为null|null
eureka.client.eureka-server-u-r-l-context|表示eureka注册中心的路径，如果配置为eureka，则为http://ip:port/eureka/,在eureka的配置文件中加入此配置表示eureka作为客户端向注册中心注册，从而构成eureka集群。此配置只有在eureka服务器ip地址列表是在DNS中才会用到，默认为null|null
eureka.client.fetch-registry|客户端是否获取eureka服务器注册表上的注册信息，默认为true|true
eureka.client.filter-only-up-instances|是否过滤掉非up实例，默认为true|true
eureka.client.g-zip-content|当服务端支持压缩的情况下，是否支持从服务端获取的信息进行压缩。默认为true||
eureka.client.property-resolver|属性解析器||
eureka.client.proxy-host|获取eureka server 的代理主机名|null
eureka.client.proxy-password|获取eureka server 的代理主机密码|null
eureka.client.proxy-port|获取eureka server 的代理主机端口|null
eureka.client.proxy-user-name|获取eureka server 的代理用户名|null
eureka.client.region|获取此实例所在的区域（在AWS数据中心中使用）。|us-east-1
eureka.client.should-enforce-registration-at-init|client在初始化阶段是否强行注册到注册中心|false
eureka.client.should-unregister-on-shutdown|client在shutdown情况下，是否显示从注册中心注销|true

