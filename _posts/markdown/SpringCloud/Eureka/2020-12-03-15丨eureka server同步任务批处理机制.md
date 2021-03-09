---
layout:     post
title:      "Eureka-15丨eureka server同步任务批处理机制"
date:       2020-12-03 23:35:36
author:     "jiefang"
header-style: text
tags:
    - Eureka
    - SpringCloud
---
# eureka server同步任务批处理机制
![eureka server同步任务批处理](https://s3.ax1x.com/2020/12/03/D7qE1U.png)

## 向队列中放入任务
`ApplicationResource#addInstance()-->PeerAwareInstanceRegistryImpl#register()-->PeerAwareInstanceRegistryImpl#replicateToPeers-->PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers-->PeerEurekaNode#register-->TaskDispatcher#process()`

```java
public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            },
            expiryTime);
}
//创建任务调度程序
TaskDispatchers#createBatchingTaskDispatcher(String id,int maxBufferSize,int workloadSize, int workerCount, long maxBatchingDelay, long congestionRetryDelayMs,long     networkFailureRetryMs,TaskProcessor<T> taskProcessor) {
    final AcceptorExecutor<ID, T> acceptorExecutor = new AcceptorExecutor<>(
            id, maxBufferSize, workloadSize, maxBatchingDelay, congestionRetryDelayMs, networkFailureRetryMs
    );
    final TaskExecutors<ID, T> taskExecutor = TaskExecutors.batchExecutors(id, workerCount, taskProcessor, acceptorExecutor);
    return new TaskDispatcher<ID, T>() {
        //提交任务到acceptorQueue
        @Override
        public void process(ID id, T task, long expiryTime) {
            acceptorExecutor.process(id, task, expiryTime);
        }
        @Override
        public void shutdown() {
            acceptorExecutor.shutdown();
            taskExecutor.shutdown();
        }
    };
}
//用于接收写入任务的队列
private final BlockingQueue<TaskHolder<ID, T>> acceptorQueue = new LinkedBlockingQueue<>();

AcceptorExecutor#process(ID id, T task, long expiryTime) {
    acceptorQueue.add(new TaskHolder<ID, T>(id, task, expiryTime));
    acceptedTasks++;
}
```
## 拆分队列


```java
//待处理任务的id
private final Deque<ID> processingOrder = new LinkedList<>();
//待处理任务
private final Map<ID, TaskHolder<ID, T>> pendingTasks = new HashMap<>();


class AcceptorRunner implements Runnable {
    @Override
    public void run() {
        long scheduleTime = 0;
        while (!isShutdown.get()) {
            try {
                drainInputQueues();
                int totalItems = processingOrder.size();
                long now = System.currentTimeMillis();
                if (scheduleTime < now) {
                    scheduleTime = now + trafficShaper.transmissionDelay();
                }
                if (scheduleTime <= now) {
                    //批处理打包
                    assignBatchWork();
                    //处理单个任务
                    assignSingleItemWork();
                }

                // If no worker is requesting data or there is a delay injected by the traffic shaper,
                // sleep for some time to avoid tight loop.
                if (totalItems == processingOrder.size()) {
                    Thread.sleep(10);
                }
            } catch (InterruptedException ex) {
                // Ignore
            } catch (Throwable e) {
                // Safe-guard, so we never exit this loop in an uncontrolled way.
                logger.warn("Discovery AcceptorThread error", e);
            }
        }
    }
    //处理输入队列
    private void drainInputQueues() throws InterruptedException {
        do {
            drainReprocessQueue();
            drainAcceptorQueue();

            if (!isShutdown.get()) {
                // If all queues are empty, block for a while on the acceptor queue
                if (reprocessQueue.isEmpty() && acceptorQueue.isEmpty() && pendingTasks.isEmpty()) {
                    TaskHolder<ID, T> taskHolder = acceptorQueue.poll(10, TimeUnit.MILLISECONDS);
                    if (taskHolder != null) {
                        appendTaskHolder(taskHolder);
                    }
                }
            }
        } while (!reprocessQueue.isEmpty() || !acceptorQueue.isEmpty() || pendingTasks.isEmpty());
    }
    //清空重新处理队列
    private void drainReprocessQueue() {
        long now = System.currentTimeMillis();
        while (!reprocessQueue.isEmpty() && !isFull()) {
            TaskHolder<ID, T> taskHolder = reprocessQueue.pollLast();
            ID id = taskHolder.getId();
            if (taskHolder.getExpiryTime() <= now) {
                expiredTasks++;
            } else if (pendingTasks.containsKey(id)) {
                overriddenTasks++;
            } else {
                pendingTasks.put(id, taskHolder);
                processingOrder.addFirst(id);
            }
        }
        if (isFull()) {
            queueOverflows += reprocessQueue.size();
            reprocessQueue.clear();
        }
    }
    //处理接收任务队列中所有数据
    private void drainAcceptorQueue() {
        while (!acceptorQueue.isEmpty()) {
            appendTaskHolder(acceptorQueue.poll());
        }
    }
    //放入pendingTasks和processingOrder
    private void appendTaskHolder(TaskHolder<ID, T> taskHolder) {
        if (isFull()) {
            pendingTasks.remove(processingOrder.poll());
            queueOverflows++;
        }
        TaskHolder<ID, T> previousTask = pendingTasks.put(taskHolder.getId(), taskHolder);
        if (previousTask == null) {
            processingOrder.add(taskHolder.getId());
        } else {
            overriddenTasks++;
        }
    }    
}
```
## 取待处理队列中的数据打包
```java
//queue中放入打包好的批处理任务
private final BlockingQueue<List<TaskHolder<ID, T>>> batchWorkQueue = new LinkedBlockingQueue<>();

//打包从待处理任务队列中获取放入batchWorkQueue
AcceptorRunner#assignBatchWork() {
    if (hasEnoughTasksForNextBatch()) {
        if (batchWorkRequests.tryAcquire(1)) {
            long now = System.currentTimeMillis();
            //len最大250
            int len = Math.min(maxBatchingSize, processingOrder.size());
            List<TaskHolder<ID, T>> holders = new ArrayList<>(len);
            //若holders里的数量小于250且待处理队列不为空
            while (holders.size() < len && !processingOrder.isEmpty()) {
                ID id = processingOrder.poll();
                TaskHolder<ID, T> holder = pendingTasks.remove(id);
                if (holder.getExpiryTime() > now) {
                    holders.add(holder);
                } else {
                    expiredTasks++;
                }
            }
            if (holders.isEmpty()) {
                batchWorkRequests.release();
            } else {
                batchSizeMetric.record(holders.size(), TimeUnit.MILLISECONDS);
                //打包放入
                batchWorkQueue.add(holders);
            }
        }
    }
}
```
## 批处理打包的queue
```java
//线程处理打包好的queue
static class BatchWorkerRunnable<ID, T> extends WorkerRunnable<ID, T> {
    BatchWorkerRunnable(String workerName,
                        AtomicBoolean isShutdown,
                        TaskExecutorMetrics metrics,
                        TaskProcessor<T> processor,
                        AcceptorExecutor<ID, T> acceptorExecutor) {
        super(workerName, isShutdown, metrics, processor, acceptorExecutor);
    }

    @Override
    public void run() {
        try {
            while (!isShutdown.get()) {
                List<TaskHolder<ID, T>> holders = getWork();
                metrics.registerExpiryTimes(holders);

                List<T> tasks = getTasksOf(holders);
                //处理任务
                ProcessingResult result = processor.process(tasks);
                switch (result) {
                    case Success:
                        break;
                    case Congestion:
                    case TransientError:
                        taskDispatcher.reprocess(holders, result);
                        break;
                    case PermanentError:
                        logger.warn("Discarding {} tasks of {} due to permanent error", holders.size(), workerName);
                }
                metrics.registerTaskResult(result, tasks.size());
            }
        } catch (InterruptedException e) {
            // Ignore
        } catch (Throwable e) {
            // Safe-guard, so we never exit this loop in an uncontrolled way.
            logger.warn("Discovery WorkerThread error", e);
        }
    }

    private List<TaskHolder<ID, T>> getWork() throws InterruptedException {
        BlockingQueue<List<TaskHolder<ID, T>>> workQueue = taskDispatcher.requestWorkItems();
        List<TaskHolder<ID, T>> result;
        do {
            result = workQueue.poll(1, TimeUnit.SECONDS);
        } while (!isShutdown.get() && result == null);
        return (result == null) ? new ArrayList<>() : result;
    }

    private List<T> getTasksOf(List<TaskHolder<ID, T>> holders) {
        List<T> tasks = new ArrayList<>(holders.size());
        for (TaskHolder<ID, T> holder : holders) {
            tasks.add(holder.getTask());
        }
        return tasks;
    }
}
//发送http请求
ReplicationTaskProcessor#process(List<ReplicationTask> tasks) {
    ReplicationList list = createReplicationListOf(tasks);
    try {
        EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
        ...
    } catch (Throwable e) {
        ...
    }
    return ProcessingResult.Success;
}
//直接打包好的数据批量http发送到其它eureka server
Jersey2ReplicationClient#submitBatchUpdates(ReplicationList replicationList) {
        Response response = null;
        try {
            //路径是peerreplication/batch/
            response = jerseyClient.target(serviceUrl)
                    .path(PeerEurekaNode.BATCH_URL_PATH)
                    .request(MediaType.APPLICATION_JSON_TYPE)
                    .post(Entity.json(replicationList));
            if (!isSuccess(response.getStatus())) {
                return anEurekaHttpResponse(response.getStatus(), ReplicationListResponse.class).build();
            }
            ReplicationListResponse batchResponse = response.readEntity(ReplicationListResponse.class);
            return anEurekaHttpResponse(response.getStatus(), batchResponse).type(MediaType.APPLICATION_JSON_TYPE).build();
        } finally {
            if (response != null) {
                response.close();
            }
        }
}
```
## 接收批量复制事件
```java
PeerReplicationResource#batchReplication(ReplicationList replicationList) {
    try {
        ReplicationListResponse batchResponse = new ReplicationListResponse();
        for (ReplicationInstance instanceInfo : replicationList.getReplicationList()) {
            try {
                batchResponse.addResponse(dispatch(instanceInfo));
            } catch (Exception e) {
              ...
            }
        }
        return Response.ok(batchResponse).build();
    } catch (Throwable e) {
        logger.error("Cannot execute batch Request", e);
        return Response.status(Status.INTERNAL_SERVER_ERROR).build();
    }
}
//分发注册、心跳、下线等事件
PeerReplicationResource#dispatch(ReplicationInstance instanceInfo) {
    ApplicationResource applicationResource = createApplicationResource(instanceInfo);
    InstanceResource resource = createInstanceResource(instanceInfo, applicationResource);

    String lastDirtyTimestamp = toString(instanceInfo.getLastDirtyTimestamp());
    String overriddenStatus = toString(instanceInfo.getOverriddenStatus());
    String instanceStatus = toString(instanceInfo.getStatus());

    Builder singleResponseBuilder = new Builder();
    switch (instanceInfo.getAction()) {
        case Register:
            singleResponseBuilder = handleRegister(instanceInfo, applicationResource);
            break;
        case Heartbeat:
            singleResponseBuilder = handleHeartbeat(serverConfig, resource, lastDirtyTimestamp, overriddenStatus, instanceStatus);
            break;
        case Cancel:
            singleResponseBuilder = handleCancel(resource);
            break;
        case StatusUpdate:
            singleResponseBuilder = handleStatusUpdate(instanceInfo, resource);
            break;
        case DeleteStatusOverride:
            singleResponseBuilder = handleDeleteStatusOverride(instanceInfo, resource);
            break;
    }
    return singleResponseBuilder.build();
}
```
