---
layout:     post
title:      "Ribbon-05丨RestTemplate实际调用与IPing"
date:       2020-12-11 23:18:29
author:     "jiefang"
header-style: text
tags:
    - Ribbon
    - SpringCloud
---
# RestTemplate实际调用与IPing

![](https://s3.ax1x.com/2020/12/11/rAvkwV.png)

## RestTemplate实际调用过程

### LoadBalancerInterceptor#intercept()

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
    @Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null,
				"Request URI does not contain a valid hostname: " + originalUri);
         //创建LoadBalancerRequest并执行RibbonLoadBalancerClient#execute方法
		return this.loadBalancer.execute(serviceName,this.requestFactory.createRequest(request, body, execution));
	}
}
```

### RibbonLoadBalancerClient#execute()

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {
	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)throws IOException {
		return execute(serviceId, request, null);
	}
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)throws IOException {
         //从ApplicationContext中获取负载均衡器，默认为ZoneAwareLoadBalancer
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
         //通过负载均衡器的负载均衡算法获取一个Server
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
		
		return execute(serviceId, ribbonServer, request);
	}
	@Override
	public <T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException {
		Server server = null;
		if (serviceInstance instanceof RibbonServer) {
			server = ((RibbonServer) serviceInstance).getServer();
		}
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
		RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);
		try {
             //执行真正的远程调用
			T returnVal = request.apply(serviceInstance);
			statsRecorder.recordStats(returnVal);
			return returnVal;
		}catch (IOException ex) {
			statsRecorder.recordStats(ex);
			throw ex;
		}catch (Exception ex) {
			statsRecorder.recordStats(ex);
			ReflectionUtils.rethrowRuntimeException(ex);
		}
		return null;
	}    
}
```

### LoadBalancerRequest#apply()

实际调用的是其子类的方法，为`LoadBalancerRequestFactory`的匿名内部类。

```java
public class LoadBalancerRequestFactory {
     //创建LoadBalancerRequest时实现的匿名内部类
	public LoadBalancerRequest<ClientHttpResponse> createRequest(
			final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) {
		return instance -> {
			HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
					this.loadBalancer);
			if (this.transformers != null) {
				for (LoadBalancerRequestTransformer transformer : this.transformers) {
					serviceRequest = transformer.transformRequest(serviceRequest,
							instance);
				}
			}
             //实际调用http请求
			return execution.execute(serviceRequest, body);
		};
	}
}
```

## execution来源

![](https://s3.ax1x.com/2020/12/11/rESefS.png)

### InterceptingClientHttpRequest.InterceptingRequestExecution#execute()

```java
private class InterceptingRequestExecution implements ClientHttpRequestExecution {
    private final Iterator<ClientHttpRequestInterceptor> iterator;
    public InterceptingRequestExecution() {
        this.iterator = interceptors.iterator();
    }
    @Override
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            return nextInterceptor.intercept(request, body, this);
        }else {
            HttpMethod method = request.getMethod();
            Assert.state(method != null, "No standard HTTP method");
            //URL转换
            ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
            request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
            if (body.length > 0) {
                if (delegate instanceof StreamingHttpOutputMessage) {
                    StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                    streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
                }
                else {
                    StreamUtils.copy(body, delegate.getBody());
                }
            }
            //真正的远程调用
            return delegate.execute();
        }
    }
}
```

## URL转换

### ServiceRequestWrapper#getURI()

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {
	@Override
	public URI getURI() {
		URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
		return uri;
	}
}
```

### RibbonLoadBalancerClient#reconstructURI()

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {
		@Override
	public URI reconstructURI(ServiceInstance instance, URI original) {
		Assert.notNull(instance, "instance can not be null");
		String serviceId = instance.getServiceId();
		RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
		URI uri;
		Server server;
		if (instance instanceof RibbonServer) {
			RibbonServer ribbonServer = (RibbonServer) instance;
			server = ribbonServer.getServer();
             //获取uri
			uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
		}
		else {
			server = new Server(instance.getScheme(), instance.getHost(),
					instance.getPort());
			IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
			ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
			uri = updateToSecureConnectionIfNeeded(original, clientConfig,
					serverIntrospector, server);
		}
         //对uri进行转换
		return context.reconstructURIWithServer(server, uri);
	}
}
```
### AbstractClientHttpRequest#execute()

```java
@Override
public final ClientHttpResponse execute() throws IOException {
	assertNotExecuted();
	ClientHttpResponse result = executeInternal(this.headers);
	this.executed = true;
	return result;
}
```
### SimpleStreamingClientHttpRequest#executeInternal
使用`spring-web.jar`包里的普通http请求处理类。
```java
@Override
protected ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException {
	try {
		if (this.body != null) {
			this.body.close();
		}
		else {
			SimpleBufferingClientHttpRequest.addHeaders(this.connection, headers);
			this.connection.connect();
			// Immediately trigger the request in a no-output scenario as well
			this.connection.getResponseCode();
		}
	}
	catch (IOException ex) {
		// ignore
	}
	return new SimpleClientHttpResponse(this.connection);
}
```
## ribbon默认IPing
### DummyPing
`RibbonClientConfiguration`注入的`DummyPing`不起作用.

```
public class RibbonClientConfiguration {
    @Bean
	@ConditionalOnMissingBean
	public IPing ribbonPing(IClientConfig config) {
		if (this.propertiesFactory.isSet(IPing.class, name)) {
			return this.propertiesFactory.get(IPing.class, config, name);
		}
		return new DummyPing();
	}
}
```
### NIWSDiscoveryPing实例化
真正起作用的是`EurekaRibbonClientConfiguration`注入的`NIWSDiscoveryPing`。
```
public class EurekaRibbonClientConfiguration {
    @Bean
	@ConditionalOnMissingBean
	public IPing ribbonPing(IClientConfig config) {
		if (this.propertiesFactory.isSet(IPing.class, serviceId)) {
			return this.propertiesFactory.get(IPing.class, config, serviceId);
		}
		NIWSDiscoveryPing ping = new NIWSDiscoveryPing();
		ping.initWithNiwsConfig(config);
		return ping;
	}
}
```
### IPing定时调度
`ZoneAwareLoadBalancer`初始化对`IPing`进行定时调度。

```java
public ZoneAwareLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping, serverList, filter, serverListUpdater);
}
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping);
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    restOfInit(clientConfig);
}
public BaseLoadBalancer(IClientConfig config, IRule rule, IPing ping) {
    initWithConfig(config, rule, ping, createLoadBalancerStatsFromConfig(config));
}
BaseLoadBalancer#initWithConfig(IClientConfig clientConfig, IRule rule, IPing ping, LoadBalancerStats stats) {
    this.config = clientConfig;
    String clientName = clientConfig.getClientName();
    this.name = clientName;
    int pingIntervalTime = Integer.parseInt(""
            + clientConfig.getProperty(CommonClientConfigKey.NFLoadBalancerPingInterval,Integer.parseInt("30")));
    int maxTotalPingTime = Integer.parseInt(""
            + clientConfig.getProperty(CommonClientConfigKey.NFLoadBalancerMaxTotalPingTime,
                    Integer.parseInt("2")));
    //设置ping时间周期为30S
    setPingInterval(pingIntervalTime);
    setMaxTotalPingTime(maxTotalPingTime);
    
    setRule(rule);
    //设置Timer定时调度
    setPing(ping);

    setLoadBalancerStats(stats);
    rule.setLoadBalancer(this);
    if (ping instanceof AbstractLoadBalancerPing) {
        ((AbstractLoadBalancerPing) ping).setLoadBalancer(this);
    }
    logger.info("Client: {} instantiated a LoadBalancer: {}", name, this);
    boolean enablePrimeConnections = clientConfig.get(
            CommonClientConfigKey.EnablePrimeConnections, DefaultClientConfigImpl.DEFAULT_ENABLE_PRIME_CONNECTIONS);

    if (enablePrimeConnections) {
        this.setEnablePrimingConnections(true);
        PrimeConnections primeConnections = new PrimeConnections(
                this.getName(), clientConfig);
        this.setPrimeConnections(primeConnections);
    }
    init();
}
```
### BaseLoadBalancer#setPing
```java
BaseLoadBalancer#setPing(IPing ping) {
    if (ping != null) {
        if (!ping.equals(this.ping)) {
            this.ping = ping;
            setupPingTask(); // since ping data changed
        }
    } else {
        this.ping = null;
        // cancel the timer task
        lbTimer.cancel();
    }
}
BaseLoadBalancer#setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
        lbTimer.cancel();
    }
    //Timer定时调度
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
            true);
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    forceQuickPing();
}
```
### PingTask
```java
class PingTask extends TimerTask {
    public void run() {
        try {
        	new Pinger(pingStrategy).runPinger();
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Error pinging", name, e);
        }
    }
}
```
### Pinger
```java
public void runPinger() throws Exception {
    if (!pingInProgress.compareAndSet(false, true)) { 
        return; // Ping in progress - nothing to do
    }
    Server[] allServers = null;
    boolean[] results = null;

    Lock allLock = null;
    Lock upLock = null;

    try {
        allLock = allServerLock.readLock();
        allLock.lock();
        allServers = allServerList.toArray(new Server[allServerList.size()]);
        allLock.unlock();

        int numCandidates = allServers.length;
        //调用IPing实现ping每一个服务实例，返回结果
        results = pingerStrategy.pingServers(ping, allServers);

        final List<Server> newUpList = new ArrayList<Server>();
        final List<Server> changedServers = new ArrayList<Server>();

        for (int i = 0; i < numCandidates; i++) {
            boolean isAlive = results[i];
            Server svr = allServers[i];
            boolean oldIsAlive = svr.isAlive();

            svr.setAlive(isAlive);

            if (oldIsAlive != isAlive) {
                changedServers.add(svr);
                logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
            		name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
            }

            if (isAlive) {
                newUpList.add(svr);
            }
        }
        upLock = upServerLock.writeLock();
        upLock.lock();
        upServerList = newUpList;
        upLock.unlock();

        notifyServerStatusChangeListener(changedServers);
    } finally {
        pingInProgress.set(false);
    }
}
```
### SerialPingStrategy
```java
private static class SerialPingStrategy implements IPingStrategy {
    @Override
    public boolean[] pingServers(IPing ping, Server[] servers) {
        int numCandidates = servers.length;
        boolean[] results = new boolean[numCandidates];
        logger.debug("LoadBalancer:  PingTask executing [{}] servers configured", numCandidates);

        for (int i = 0; i < numCandidates; i++) {
            results[i] = false; /* Default answer is DEAD. */
            try {
                if (ping != null) {
                    results[i] = ping.isAlive(servers[i]);
                }
            } catch (Exception e) {
                logger.error("Exception while pinging Server: '{}'", servers[i], e);
            }
        }
        return results;
    }
}
```
### NIWSDiscoveryPing#isAlive
简单的根据每个服务实例的状态是否存活来判断。
```java
NIWSDiscoveryPing#isAlive(Server server) {
    boolean isAlive = true;
    if (server!=null && server instanceof DiscoveryEnabledServer){
        DiscoveryEnabledServer dServer = (DiscoveryEnabledServer)server;	            
        InstanceInfo instanceInfo = dServer.getInstanceInfo();
        if (instanceInfo!=null){	                
            InstanceStatus status = instanceInfo.getStatus();
            if (status!=null){
                isAlive = status.equals(InstanceStatus.UP);
            }
        }
    }
    return isAlive;
}
```


