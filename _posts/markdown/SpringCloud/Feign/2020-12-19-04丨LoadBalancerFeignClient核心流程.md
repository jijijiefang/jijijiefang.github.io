---
layout:     post
title:      "Feign-04丨LoadBalancerFeignClient核心流程"
date:       2020-12-19 21:05:34
author:     "jiefang"
header-style: text
tags:
    - Feign
    - SpringCloud
---
# LoadBalancerFeignClient核心流程

![](https://s3.ax1x.com/2020/12/19/rUiAMt.png)

## 自动装载

### FeignRibbonClientAutoConfiguration

```java
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@ConditionalOnProperty(value = "spring.cloud.loadbalancer.ribbon.enabled",
		matchIfMissing = true)
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(FeignAutoConfiguration.class)
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {
	//创建FeignLoadBalancer的工厂类
	@Bean
	@Primary
	@ConditionalOnMissingBean
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	public CachingSpringLoadBalancerFactory cachingLBClientFactory(
			SpringClientFactory factory) {
		return new CachingSpringLoadBalancerFactory(factory);
	}
	//创建带有重试策略FeignLoadBalancer的工厂类
	@Bean
	@Primary
	@ConditionalOnMissingBean
	@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
	public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
			SpringClientFactory factory, LoadBalancedRetryFactory retryFactory) {
		return new CachingSpringLoadBalancerFactory(factory, retryFactory);
	}
	...
}
```

### DefaultFeignLoadBalancedConfiguration

```java
@Configuration(proxyBeanMethods = false)
class DefaultFeignLoadBalancedConfiguration {
	//使用默认Client自动装配LoadBalancerFeignClient
	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
			SpringClientFactory clientFactory) {
		return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
				clientFactory);
	}
}
```

## SynchronousMethodHandler#executeAndDecode

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
    Request request = targetRequest(template);
    Response response;
    long start = System.nanoTime();
    try {
        //LoadBalancerFeignClient调用
        response = client.execute(request, options);
    } catch (IOException e) {
       ...
    }
    boolean shouldClose = true;
    try {
        if (response.status() >= 200 && response.status() < 300) {
            if (void.class == metadata.returnType()) {
                return null;
            } else {
                return decode(response);
            }
        } else if (decode404 && response.status() == 404) {
            //使用默认的ResponseEntityDecoder解析Response
            return decoder.decode(response, metadata.returnType());
        } else {
            throw errorDecoder.decode(metadata.configKey(), response);
        }
    } catch (IOException e) {
       ...
    }
}
```

## LoadBalancerFeignClient

```java
public class LoadBalancerFeignClient implements Client {
    static final Request.Options DEFAULT_OPTIONS = new Request.Options();
	//使用默认的Client或ApacheHttpClient或OkHttpClient
	private final Client delegate;
	//FeignRibbonClientAutoConfiguration加载
	private CachingSpringLoadBalancerFactory lbClientFactory;
	//RibbonAutoConfiguration加载
	private SpringClientFactory clientFactory;
	
	public LoadBalancerFeignClient(Client delegate,
			CachingSpringLoadBalancerFactory lbClientFactory,
			SpringClientFactory clientFactory) {
		this.delegate = delegate;
		this.lbClientFactory = lbClientFactory;
		this.clientFactory = clientFactory;
	}
}
```

### execute()

```java
@Override
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        URI asUri = URI.create(request.url());
        String clientName = asUri.getHost();
        //去除URL里的https://或http://
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
            this.delegate, request, uriWithoutHost);

        IClientConfig requestConfig = getClientConfig(options, clientName);
        //创建FeignLoadBalancer并执行其父类AbstractLoadBalancerAwareClient的executeWithLoadBalancer方法
        return lbClient(clientName).executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
    }catch (ClientException e) {
        IOException io = findIOException(e);
        if (io != null) {
            throw io;
        }
        throw new RuntimeException(e);
    }
}
```

### lbClient()

```java
private FeignLoadBalancer lbClient(String clientName) {
    return this.lbClientFactory.create(clientName);
}
```

## CachingSpringLoadBalancerFactory

```java
public class CachingSpringLoadBalancerFactory {
    protected final SpringClientFactory factory;
	protected LoadBalancedRetryFactory loadBalancedRetryFactory = null;
	private volatile Map<String, FeignLoadBalancer> cache = new ConcurrentReferenceHashMap<>();
	public CachingSpringLoadBalancerFactory(SpringClientFactory factory) {
		this.factory = factory;
	}
	public CachingSpringLoadBalancerFactory(SpringClientFactory factory,
			LoadBalancedRetryFactory loadBalancedRetryPolicyFactory) {
		this.factory = factory;
		this.loadBalancedRetryFactory = loadBalancedRetryPolicyFactory;
	}
	//获取或创建FeignLoadBalancer
	public FeignLoadBalancer create(String clientName) {
		FeignLoadBalancer client = this.cache.get(clientName);
		if (client != null) {
			return client;
		}
		IClientConfig config = this.factory.getClientConfig(clientName);
		ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
		ServerIntrospector serverIntrospector = this.factory.getInstance(clientName,
				ServerIntrospector.class);
        //重试策略的FeignLoadBalancer或普通FeignLoadBalancer
		client = this.loadBalancedRetryFactory != null
				? new RetryableFeignLoadBalancer(lb, config, serverIntrospector,
						this.loadBalancedRetryFactory)
				: new FeignLoadBalancer(lb, config, serverIntrospector);
		this.cache.put(clientName, client);
		return client;
	}
}
```

## AbstractLoadBalancerAwareClient

```java
public abstract class AbstractLoadBalancerAwareClient<S extends ClientRequest, T extends IResponse> extends LoadBalancerContext implements IClient<S, T>, IClientConfigAware {
    public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

        try {
            return command.submit(
                new ServerOperation<T>() {
                    @Override
                    public Observable<T> call(Server server) {
                        //构建最后完整的URL
                        URI finalUri = reconstructURIWithServer(server, request.getUri());
                        //组装Http请求的参数包括Entity、Headers、QueryParams等
                        S requestForServer = (S) request.replaceUri(finalUri);
                        try {
                            return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                        } 
                        catch (Exception e) {
                            return Observable.error(e);
                        }
                    }
                })
                .toBlocking()
                .single();
        } catch (Exception e) {
            Throwable t = e.getCause();
            if (t instanceof ClientException) {
                throw (ClientException) t;
            } else {
                throw new ClientException(e);
            }
        }
    }
}
```

### LoadBalancerCommand

```java
public class LoadBalancerCommand<T> {
    public Observable<T> submit(final ServerOperation<T> operation) {
        ...
        //通过LoadBalancer选择一个Server
        Observable<T> o = (server == null ? selectServer() : Observable.just(server))
                    .concatMap(new Func1<Server, Observable<T>>() {

        }
    }
    //通过LoadBalancer选择一个Server
    private Observable<Server> selectServer() {
        return Observable.create(new OnSubscribe<Server>() {
            @Override
            public void call(Subscriber<? super Server> next) {
                 try{
                     Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                     next.onNext(server);
                     next.onCompleted();
                 }catch (Exception e) {
                     next.onError(e);
                 }
            }
        });
    }
}                      
```

### LoadBalancerContext

```java
public class LoadBalancerContext implements IClientConfigAware {
    	//通过Ribbon集成的LoadBalancer选择Server
        public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
		...
        ILoadBalancer lb = getLoadBalancer();
        ...
        return new Server(host, port);
    }
}
```

### FeignLoadBalancer

```java
public class FeignLoadBalancer extends AbstractLoadBalancerAwareClient<FeignLoadBalancer.RibbonRequest, FeignLoadBalancer.RibbonResponse> {
    @Override
	public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
			throws IOException {
		Request.Options options;
		if (configOverride != null) {
			RibbonProperties override = RibbonProperties.from(configOverride);
			options = new Request.Options(override.connectTimeout(this.connectTimeout),
					override.readTimeout(this.readTimeout));
		}
		else {
			options = new Request.Options(this.connectTimeout, this.readTimeout);
		}
        //使用Default、ApacheHttpClient或执行Http请求并返回Response
		Response response = request.client().execute(request.toRequest(), options);
		return new RibbonResponse(request.getUri(), response);
	}
}
```
