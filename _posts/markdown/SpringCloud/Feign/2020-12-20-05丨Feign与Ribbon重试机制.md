---
layout:     post
title:      "Feign-05丨Feign与Ribbon重试机制"
date:       2020-12-20 17:02:16
author:     "jiefang"
header-style: text
tags:
    - Feign
    - SpringCloud
---
# Feign与Ribbon重试机制

## yml配置

```yml
spring:
  cloud:
	loadbalancer:
  		retry:
    		enabled: true
ribbon:
  ConnectTimeout: 1000--连接超时设置
  ReadTimeout: 1000--读取超时设置
  OkToRetryOnAllOperations: true--无论请求超时异常等都重试
  MaxAutoRetries: 1-当前选择的服务实例重试1次
  MaxAutoRetriesNextServer: 3-选择其它服务实例重试3次
```

## 源码

### LoadBalancerCommand

Ribbon重试策略

```java
public class LoadBalancerCommand<T> {
        public Observable<T> submit(final ServerOperation<T> operation) {
        final ExecutionInfoContext context = new ExecutionInfoContext();
        
        if (listenerInvoker != null) {
            try {
                listenerInvoker.onExecutionStart();
            } catch (AbortExecutionException e) {
                return Observable.error(e);
            }
        }
		//MaxAutoRetries
        final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
        //MaxAutoRetriesNextServer    
        final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

        // Use the load balancer
        Observable<T> o = 
                (server == null ? selectServer() : Observable.just(server))
                .concatMap(new Func1<Server, Observable<T>>() {
                    @Override
                    // Called for each server being selected
                    public Observable<T> call(Server server) {
                        context.setServer(server);
                        final ServerStats stats = loadBalancerContext.getServerStats(server);
                        
                        // Called for each attempt and retry
                        Observable<T> o = Observable
                                .just(server)
                                .concatMap(new Func1<Server, Observable<T>>() {
                                    @Override
                                    public Observable<T> call(final Server server) {
                                        context.incAttemptCount();
                                        loadBalancerContext.noteOpenConnection(stats);
                                        
                                        if (listenerInvoker != null) {
                                            try {
                                                listenerInvoker.onStartWithServer(context.toExecutionInfo());
                                            } catch (AbortExecutionException e) {
                                                return Observable.error(e);
                                            }
                                        }
                                        
                                        final Stopwatch tracer = loadBalancerContext.getExecuteTracer().start();
                                        
                                        return operation.call(server).doOnEach(new Observer<T>() {
                                            private T entity;
                                            @Override
                                            public void onCompleted() {
                                                recordStats(tracer, stats, entity, null);
                                            }

                                            @Override
                                            public void onError(Throwable e) {
                                                ...
                                            }

                                            @Override
                                            public void onNext(T entity) {
                                                ...
                                            }                            
                                            
                                            private void recordStats(Stopwatch tracer, ServerStats stats, Object entity, Throwable exception) {
                                                ...
                                            }
                                        });
                                    }
                                });
                        //当前服务重试
                        if (maxRetrysSame > 0) 
                            o = o.retry(retryPolicy(maxRetrysSame, true));
                        return o;
                    }
                });
        //其它服务重试    
        if (maxRetrysNext > 0 && server == null) 
            o = o.retry(retryPolicy(maxRetrysNext, false));
        //重试后失败最终抛出异常
        return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
            @Override
            public Observable<T> call(Throwable e) {
                ...
            }
        });
    }
}
```

### SynchronousMethodHandler#invoke

Feign重试策略。

```java
final class SynchronousMethodHandler implements MethodHandler {
    @Override
    public Object invoke(Object[] argv) throws Throwable {
        RequestTemplate template = buildTemplateFromArgs.create(argv);
        Retryer retryer = this.retryer.clone();
        while (true) {
            try {
                return executeAndDecode(template);
            } catch (RetryableException e) {
                retryer.continueOrPropagate(e);
                if (logLevel != Logger.Level.NONE) {
                    logger.logRetry(metadata.configKey(), logLevel);
                }
                continue;
            }
        }
    }
}
```

### Retryer

重试策略分为默认重试策略和不重试。

```java
public interface Retryer extends Cloneable {

    void continueOrPropagate(RetryableException e);

    Retryer clone();

    public static class Default implements Retryer {

        private final int maxAttempts;
        private final long period;
        private final long maxPeriod;
        int attempt;
        long sleptForMillis;
		//默认为最多重试5次，最大等待重试时间为1S
        public Default() {
            this(100, SECONDS.toMillis(1), 5);
        }

        public Default(long period, long maxPeriod, int maxAttempts) {
            this.period = period;
            this.maxPeriod = maxPeriod;
            this.maxAttempts = maxAttempts;
            this.attempt = 1;
        }

        // visible for testing;
        protected long currentTimeMillis() {
            return System.currentTimeMillis();
        }
		
        public void continueOrPropagate(RetryableException e) {
            if (attempt++ >= maxAttempts) {
                throw e;
            }
            long interval;
            if (e.retryAfter() != null) {
                interval = e.retryAfter().getTime() - currentTimeMillis();
                if (interval > maxPeriod) {
                    interval = maxPeriod;
                }
                if (interval < 0) {
                    return;
                }
            } else {
                interval = nextMaxInterval();
            }
            try {
                Thread.sleep(interval);
            } catch (InterruptedException ignored) {
                Thread.currentThread().interrupt();
            }
            sleptForMillis += interval;
        }
		//计算下次等待重试时间
        long nextMaxInterval() {
            long interval = (long) (period * Math.pow(1.5, attempt - 1));
            return interval > maxPeriod ? maxPeriod : interval;
        }

        @Override
        public Retryer clone() {
            return new Default(period, maxPeriod, maxAttempts);
        }
    }
	//不重试
    Retryer NEVER_RETRY = new Retryer() {
        @Override
        public void continueOrPropagate(RetryableException e) {
            throw e;
        }
        @Override
        public Retryer clone() {
            return this;
        }
    };
}
```

