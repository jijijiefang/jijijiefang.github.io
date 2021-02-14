---
layout:     post
title:      "Ribbon-02丨Ribbon与RestTemplate源码分析"
date:       2020-12-09 21:50:09
author:     "jiefang"
header-style: text
tags:
    - Ribbon
    - SpringCloud
---
# ribbon 与RestTemplate源码分析

本文主要针对 `spring-cloud-starter-netflix-ribbon` 的 **2.2.2.RELEASE** 版本进行源码的解析。

## 注解LoadBalanced

使用注解@LoadBalanced,注解在RestTemplate上。在spring-cloud-commons包中。

```java
package org.springframework.cloud.client.loadbalancer;
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}
```

## 自动装配

### LoadBalancerAutoConfiguration

`@LoadBalanced`注解使用的是`AutoConfiguration`是`LoadBalancerAutoConfiguration`。

```java
package org.springframework.cloud.client.loadbalancer;

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerProperties.class)
public class LoadBalancerAutoConfiguration {
    @LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();
    
    @Configuration(proxyBeanMethods = false)
	@Conditional(RetryMissingOrDisabledCondition.class)
	static class LoadBalancerInterceptorConfig {
		//加载负载均衡拦截器,传入的LoadBalancerClient是RibbonLoadBalancerClient
		@Bean
		public LoadBalancerInterceptor loadBalancerInterceptor(LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}
		//restTemplate设置拦截器
		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}
	}
    ...
}
```

### LoadBalancerInterceptor

负载均衡拦截器。

```java
package org.springframework.cloud.client.loadbalancer;
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;
	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer, LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}
	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}
	//拦截RestTemplate，执行方法
	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
         //此处的loadBalancer就是之前注入的RibbonLoadBalancerClient
		return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
	}
}
```

### RibbonAutoConfiguration

`LoadBalancerAutoConfiguration`在装配`LoadBalancerInterceptor`时传入的`LoadBalancerClient`，是`RibbonLoadBalancerClient`实例；而`RibbonLoadBalancerClient`使用`RibbonAutoConfiguration`进行装配。

```java
package org.springframework.cloud.netflix.ribbon;

@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
//RibbonAutoConfiguration的装配依赖于EurekaClient的自动装配
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
//RibbonAutoConfiguration的装配先于LoadBalancerAutoConfiguration的装配
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {

	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>();

	@Autowired
	private RibbonEagerLoadProperties ribbonEagerLoadProperties;

	@Bean
	public HasFeatures ribbonFeature() {
		return HasFeatures.namedFeature("Ribbon", Ribbon.class);
	}

	@Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}
	//注入负载均衡客户端实例RibbonLoadBalancerClient
	@Bean
	@ConditionalOnMissingBean(LoadBalancerClient.class)
	public LoadBalancerClient loadBalancerClient() {
		return new RibbonLoadBalancerClient(springClientFactory());
	}
	...
}
```

### RibbonClientConfiguration

```java
@Import({ HttpClientConfiguration.class, OkHttpRibbonConfiguration.class,
         RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class })
public class RibbonClientConfiguration {
    ...
     //IClientConfig的实例
	@Bean
	@ConditionalOnMissingBean
	public IClientConfig ribbonClientConfig() {
		DefaultClientConfigImpl config = new DefaultClientConfigImpl();
		config.loadProperties(this.name);
		config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
		config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
		config.set(CommonClientConfigKey.GZipPayload, DEFAULT_GZIP_PAYLOAD);
		return config;
	}
	//默认的负载均衡策略
	@Bean
	@ConditionalOnMissingBean
	public IRule ribbonRule(IClientConfig config) {
		if (this.propertiesFactory.isSet(IRule.class, name)) {
			return this.propertiesFactory.get(IRule.class, config, name);
		}
		ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
		rule.initWithNiwsConfig(config);
		return rule;
	}
	//默认的维持心跳IPing实现
	@Bean
	@ConditionalOnMissingBean
	public IPing ribbonPing(IClientConfig config) {
		if (this.propertiesFactory.isSet(IPing.class, name)) {
			return this.propertiesFactory.get(IPing.class, config, name);
		}
		return new DummyPing();
	}
	//默认的服务列表配置类
	@Bean
	@ConditionalOnMissingBean
	@SuppressWarnings("unchecked")
	public ServerList<Server> ribbonServerList(IClientConfig config) {
		if (this.propertiesFactory.isSet(ServerList.class, name)) {
			return this.propertiesFactory.get(ServerList.class, config, name);
		}
		ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
		serverList.initWithNiwsConfig(config);
		return serverList;
	}
    //定时获取服务实例列表更新
	@Bean
	@ConditionalOnMissingBean
	public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
		return new PollingServerListUpdater(config);
	}
	//默认的负载均衡实现实例
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
    ...
}
```



## RestTemplate 的调用过程

以`RestTemplate`的`getForEntity()`方法为例。

### RestTemplate#doExecute

```java
package org.springframework.web.client;
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations {
@Nullable
	protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
			@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

		Assert.notNull(url, "URI is required");
		Assert.notNull(method, "HttpMethod is required");
		ClientHttpResponse response = null;
		try {
             //创建ClientHttpRequest对象
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
			response = request.execute();
			handleResponse(url, method, response);
			return (responseExtractor != null ? responseExtractor.extractData(response) : null);
		}catch (IOException ex) {
			String resource = url.toString();
			String query = url.getRawQuery();
			resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
			throw new ResourceAccessException("I/O error on " + method.name() +
					" request for \"" + resource + "\": " + ex.getMessage(), ex);
		}finally {
			if (response != null) {
				response.close();
			}
		}
	}
}
```

### HttpAccessor#createRequest

```java
package org.springframework.http.client.support;
public abstract class HttpAccessor {
    protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
        //createRequest返回的是InterceptingClientHttpRequest
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
		initialize(request);
		if (logger.isDebugEnabled()) {
			logger.debug("HTTP " + method.name() + " " + url);
		}
		return request;
	}
}
```

### InterceptingHttpAccessor #getRequestFactory

`getRequestFactory()`最后返回的是`InterceptingClientHttpRequestFactory`。

```java
package org.springframework.http.client.support;
public abstract class InterceptingHttpAccessor extends HttpAccessor {
    @Override
	public ClientHttpRequestFactory getRequestFactory() {
         //interceptors是LoadBalancerAutoConfiguration.LoadBalancerInterceptorConfig#restTemplateCustomizer()设置的
		List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
		if (!CollectionUtils.isEmpty(interceptors)) {
			ClientHttpRequestFactory factory = this.interceptingRequestFactory;
			if (factory == null) {
				factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
				this.interceptingRequestFactory = factory;
			}
			return factory;
		}
		else {
			return super.getRequestFactory();
		}
	}
}
```

### request.execute()

request执行`execute()`方法，会调到`InterceptingClientHttpRequest`的父类`AbstractClientHttpRequest#execute()`。

```java
package org.springframework.http.client;
public abstract class AbstractClientHttpRequest implements ClientHttpRequest {
    	@Override
	public final ClientHttpResponse execute() throws IOException {
		assertNotExecuted();
        //调到子类InterceptingClientHttpRequest的executeInternal方法
		ClientHttpResponse result = executeInternal(this.headers);
		this.executed = true;
		return result;
	}
}
```

#### InterceptingClientHttpRequest#executeInternal

```java
class InterceptingClientHttpRequest extends AbstractBufferingClientHttpRequest {
		@Override
	protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
		InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
		return requestExecution.execute(this, bufferedOutput);
	}
private class InterceptingRequestExecution implements ClientHttpRequestExecution {
		private final Iterator<ClientHttpRequestInterceptor> iterator;
		public InterceptingRequestExecution() {
			this.iterator = interceptors.iterator();
		}
		@Override
		public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
             //执行LoadBalancerInterceptor拦截器intercept方法
			if (this.iterator.hasNext()) {
				ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
				return nextInterceptor.intercept(request, body, this);
			}else {
				HttpMethod method = request.getMethod();
				Assert.state(method != null, "No standard HTTP method");
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
				return delegate.execute();
			}
		}
	}    
}
```

### getLoadBalancer过程

![](https://s3.ax1x.com/2020/12/09/rCRgbT.png)

####  SpringClientFactory#getInstance

```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
    protected ILoadBalancer getLoadBalancer(String serviceId) {
        return this.clientFactory.getLoadBalancer(serviceId);
    }
}
```

#### SpringClientFactory#getLoadBalancer

```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
    //根据服务名获取负载均衡器，在RibbonClientConfiguration初始化的ZoneAwareLoadBalancer
    public ILoadBalancer getLoadBalancer(String name) {
		return getInstance(name, ILoadBalancer.class);
	}
	@Override
	public <C> C getInstance(String name, Class<C> type) {
		C instance = super.getInstance(name, type);
         //直接获取到就返回
		if (instance != null) {
			return instance;
		}
         //没有获取到ILoadBalancer，则通过IClientConfig反射生成返回
		IClientConfig config = getInstance(name, IClientConfig.class);
		return instantiateWithConfig(getContext(name), type, config);
	}    
}
```

#### NamedContextFactory#getInstance

```java
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification> implements DisposableBean, ApplicationContextAware {
	public <T> T getInstance(String name, Class<T> type) {
         //获取每个服务对应的ApplicationContext
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return context.getBean(type);
		}
		return null;
	}
    //获取每个服务对应的ApplicationContext,如果没有获取到就生成并返回
	protected AnnotationConfigApplicationContext getContext(String name) {
         //map里没有就加锁
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
                      //生成一个service对应的ApplicationContext并放入map
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}
}
```

### RibbonLoadBalancerClient

对`RestTemplate`进行拦截后，执行的真正的是`RibbonLoadBalancerClient`。

```java
package org.springframework.cloud.netflix.ribbon;
public class RibbonLoadBalancerClient implements LoadBalancerClient {
    	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
		return execute(serviceId, request, null);
	}
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
         //获取负载均衡器
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
         //通过负载均衡器及负载均衡算法选择一个对应的服务
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
         //构造RibbonServer
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));
         //调用
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

		RibbonLoadBalancerContext context = this.clientFactory
				.getLoadBalancerContext(serviceId);
		RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

		try {
			T returnVal = request.apply(serviceInstance);
			statsRecorder.recordStats(returnVal);
			return returnVal;
		}
		// catch IOException and rethrow so RestTemplate behaves correctly
		catch (IOException ex) {
			statsRecorder.recordStats(ex);
			throw ex;
		}
		catch (Exception ex) {
			statsRecorder.recordStats(ex);
			ReflectionUtils.rethrowRuntimeException(ex);
		}
		return null;
	}    
}
```
