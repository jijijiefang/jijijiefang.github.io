---
layout:     post
title:      "Hystrix-03丨Hystrix与Feign整合"
date:       2020-12-23 22:45:59
author:     "jiefang"
header-style: text
tags:
    - Feign
    - Hystrix
    - SpringCloud
---
# Hystrix与Feign整合

## 自动装配

`feign.hystrix.enabled`，feign启用Hystrix时，使用`HystrixFeign.builder`。

```java
@Configuration(proxyBeanMethods = false)
public class FeignClientsConfiguration {
    @Configuration(proxyBeanMethods = false)
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {
		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}
	}
}
```

## FeignClientFactoryBean生成代理对象

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
	<T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
        
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
			this.url += cleanPath();
            //ribbon负载均衡构建
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
        //硬编码URL构建
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}
    //使用Ribbon构建
	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
            //使用HystrixTargeter来创建代理对象
			return targeter.target(this, builder, context, target);
		}
		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}    
}
```

### HystrixTargeter

```java
class HystrixTargeter implements Targeter {
	@Override
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
			FeignContext context, Target.HardCodedTarget<T> target) {
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
				: factory.getContextId();
        //设置setterFactory
		SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
        //设置fallback或fallback工厂
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(name, context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
            //使用HystrixFeign.HystrixInvocationHandler#create()方法
			return targetWithFallbackFactory(name, context, target, builder,
					fallbackFactory);
		}
		return feign.target(target);
	}
  	
	private <T> T targetWithFallbackFactory(String feignClientName, FeignContext context,
			Target.HardCodedTarget<T> target, HystrixFeign.Builder builder,
			Class<?> fallbackFactoryClass) {
        //FeignContext中获取FallbackFactory，否则抛异常
		FallbackFactory<? extends T> fallbackFactory = (FallbackFactory<? extends T>) getFromContext(
				"fallbackFactory", feignClientName, context, fallbackFactoryClass,
				FallbackFactory.class);
		return builder.target(target, fallbackFactory);
	}    
}
```



## HystrixFeign

```java
public final class HystrixFeign {
    //HystrixFeign使用InvocationHandlerFactory创建HystrixInvocationHandler，设置FallbackFactory
    Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override
        public InvocationHandler create(Target target,
                                        Map<Method, MethodHandler> dispatch) {
          return new HystrixInvocationHandler(target, dispatch, setterFactory,
              nullableFallbackFactory);
        }
      });
      //HystrixFeign使用HystrixDelegatingContract
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
    }
}
```

## ReflectiveFeign

```java
public class ReflectiveFeign extends Feign { 
    @Override
    public <T> T newInstance(Target<T> target) {
        Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
        Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
        List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

        for (Method method : target.type().getMethods()) {
            if (method.getDeclaringClass() == Object.class) {
                continue;
            } else if (Util.isDefault(method)) {
                DefaultMethodHandler handler = new DefaultMethodHandler(method);
                defaultMethodHandlers.add(handler);
                methodToHandler.put(method, handler);
            } else {
                methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
            }
        }
        //使用的是HystrixInvocationHandler
        InvocationHandler handler = factory.create(target, methodToHandler);
        T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
                                             new Class<?>[] {target.type()}, handler);

        for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
            defaultMethodHandler.bindTo(proxy);
        }
        return proxy;
    }
}
```

## SetterFactory

```java
public interface SetterFactory {
    /**
   * Returns a hystrix setter appropriate for the given target and method
   */
    HystrixCommand.Setter create(Target<?> target, Method method);
    /**
   * Default behavior is to derive the group key from {@link Target#name()} and the command key from
   * {@link Feign#configKey(Class, Method)}.
   */
    final class Default implements SetterFactory {
        //为每个方法创建Setter
        @Override
        public HystrixCommand.Setter create(Target<?> target, Method method) {
            String groupKey = target.name();
            String commandKey = Feign.configKey(target.type(), method);
            return HystrixCommand.Setter
                .withGroupKey(HystrixCommandGroupKey.Factory.asKey(groupKey))
                .andCommandKey(HystrixCommandKey.Factory.asKey(commandKey));
        }
    }
}
```



## HystrixInvocationHandler#invoke

```java
final class HystrixInvocationHandler implements InvocationHandler {
    //为每个方法创建Setter
    static Map<Method, Setter> toSetters(SetterFactory setterFactory,
                                         Target<?> target,
                                         Set<Method> methods) {
        Map<Method, Setter> result = new LinkedHashMap<Method, Setter>();
        for (Method method : methods) {
            method.setAccessible(true);
            //Setter为每个方法对应的HystrixCommand，设置groupKey和commandKey
            result.put(method, setterFactory.create(target, method));
        }
        return result;
    }
    @Override
    public Object invoke(final Object proxy, final Method method, final Object[] args)
        throws Throwable {
        // early exit if the invoked method is from java.lang.Object
        // code is the same as ReflectiveFeign.FeignInvocationHandler
        if ("equals".equals(method.getName())) {
            try {
                Object otherHandler =
                    args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
                return equals(otherHandler);
            } catch (IllegalArgumentException e) {
                return false;
            }
        } else if ("hashCode".equals(method.getName())) {
            return hashCode();
        } else if ("toString".equals(method.getName())) {
            return toString();
        }
		//创建HystrixCommand
        HystrixCommand<Object> hystrixCommand =
            new HystrixCommand<Object>(setterMethodMap.get(method)) {
            @Override
            protected Object run() throws Exception {
                try {
                    //真正的方法调用
                    return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
                } catch (Exception e) {
                    throw e;
                } catch (Throwable t) {
                    throw (Error) t;
                }
            }
			//HystrixCommand降级方法
            @Override
            protected Object getFallback() {
                if (fallbackFactory == null) {
                    return super.getFallback();
                }
                try {
                    Object fallback = fallbackFactory.create(getExecutionException());
                    Object result = fallbackMethodMap.get(method).invoke(fallback, args);
                    if (isReturnsHystrixCommand(method)) {
                        return ((HystrixCommand) result).execute();
                    } else if (isReturnsObservable(method)) {
                        // Create a cold Observable
                        return ((Observable) result).toBlocking().first();
                    } else if (isReturnsSingle(method)) {
                        // Create a cold Observable as a Single
                        return ((Single) result).toObservable().toBlocking().first();
                    } else if (isReturnsCompletable(method)) {
                        ((Completable) result).await();
                        return null;
                    } else if (isReturnsCompletableFuture(method)) {
                        return ((Future) result).get();
                    } else {
                        return result;
                    }
                } catch (IllegalAccessException e) {
                    // shouldn't happen as method is public due to being an interface
                    throw new AssertionError(e);
                } catch (InvocationTargetException | ExecutionException e) {
                    // Exceptions on fallback are tossed by Hystrix
                    throw new AssertionError(e.getCause());
                } catch (InterruptedException e) {
                    // Exceptions on fallback are tossed by Hystrix
                    Thread.currentThread().interrupt();
                    throw new AssertionError(e.getCause());
                }
            }
        };

        if (Util.isDefault(method)) {
            return hystrixCommand.execute();
        } else if (isReturnsHystrixCommand(method)) {
            return hystrixCommand;
        } else if (isReturnsObservable(method)) {
            // Create a cold Observable
            return hystrixCommand.toObservable();
        } else if (isReturnsSingle(method)) {
            // Create a cold Observable as a Single
            return hystrixCommand.toObservable().toSingle();
        } else if (isReturnsCompletable(method)) {
            return hystrixCommand.toObservable().toCompletable();
        } else if (isReturnsCompletableFuture(method)) {
            return new ObservableCompletableFuture<>(hystrixCommand);
        }
        return hystrixCommand.execute();
    }
}
```

