---
layout:     post
title:      "Java多线程-27丨JUC-ScheduledThreadPoolExecutor"
date:       2019-12-24 20:01:22
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# ScheduledThreadPoolExecutor

## 简介
Java注释
>A {@link ThreadPoolExecutor} that can additionally schedule commands to run after a given delay, or to execute periodically. This class is preferable to {@link java.util.Timer} when multiple worker threads are needed, or when the additional flexibility or capabilities of {@link ThreadPoolExecutor} (which this class extends) are required.

翻译
>可以安排命令在给定延迟后运行或定期执行的`ThreadPoolExecutor`。当需要多个工作线程时，或者需要`ThreadPoolExecutor`的附加灵活性或功能时，此类比`java.util.Timer`更可取。

`ScheduledThreadPoolExecutor`继承自`ThreadPoolExecutor`，为任务提供延迟或周期执行，属于线程池的一种。

### 类图

![image](https://s2.ax1x.com/2019/12/24/lCxmgH.png)

## 源码


### 属性
```
    //如果应在关机时取消/取消定期任务，则为False。
    private volatile boolean continueExistingPeriodicTasksAfterShutdown;
    //如果应在关闭时取消非定期任务，则为False。
    private volatile boolean executeExistingDelayedTasksAfterShutdown = true;
    //如果ScheduledFutureTask.cancel应该从队列中删除，则为True
    private volatile boolean removeOnCancel = false;
    //为相同延时的任务提供的顺序编号，保证任务之间的FIFO顺序
    private static final AtomicLong sequencer = new AtomicLong();    
```
### 内部类
#### DelayedWorkQueue
```
//为存储周期或延迟任务定义的延迟队列，继承了 AbstractQueue，实现BlockingQueue 接口。它内部只允许存储 RunnableScheduledFuture 类型的任务。
static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
    //初始容量
    private static final int INITIAL_CAPACITY = 16;
    //初始队列
    private RunnableScheduledFuture<?>[] queue =
        new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    //锁
    private final ReentrantLock lock = new ReentrantLock();
    //元素数量
    private int size = 0;
    //队列头等待任务的线程
    private Thread leader = null;
    //当队列头上有新任务可用时或新线程可能成为领导者时，发出条件信号。
    private final Condition available = lock.newCondition();
    //ThreadPoolExecutor.getTask()会调到take方法
    public RunnableScheduledFuture<?> take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        try {
            for (;;) {
                //堆顶任务
                RunnableScheduledFuture<?> first = queue[0];
                //任务为空，阻塞
                if (first == null)
                    available.await();
                else {
                    //返回剩余延迟时间
                    long delay = first.getDelay(NANOSECONDS);
                    //小于等于0，说明这个任务到时间了，可以从队列中出队了
                    if (delay <= 0)
                        //出队，堆化
                        return finishPoll(first);
                    //没到时间
                    first = null; // don't retain ref while waiting
                    //如果前面有线程在等待，直接进入等待
                    if (leader != null)
                        available.await();
                    else {
                        //否则当前线程是leader
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            //等待延时时间，然后唤醒
                            available.awaitNanos(delay);
                        } finally {
                            //leader置空
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            //唤醒下一个等待的线程
            if (leader == null && queue[0] != null)
                available.signal();
            //解锁
            lock.unlock();
        }
    }    
}
```
#### ScheduledFutureTask
```
private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {
    //FIFO的任务编号
    private final long sequenceNumber;
    //延迟时间
    private long time;
    //重复任务的周期（以纳秒为单位）。正表示固定汇率执行。负值表示固定延迟执行。值为0表示非重复任务。
    private final long period;
    //实际任务
    RunnableScheduledFuture<V> outerTask = this;
    //队列索引
    int heapIndex;
    
    //覆盖FutureTask的run方法
    public void run() {
        //是否周期性任务
        boolean periodic = isPeriodic();
        //线程池状态判断,线程池关闭策略
        if (!canRunInCurrentRunState(periodic))
            //取消
            cancel(false);
        //一次性任务调用FutureTask的run方法
        else if (!periodic)
            ScheduledFutureTask.super.run();
        //周期性任务调用FutureTask的runAndReset方法
        else if (ScheduledFutureTask.super.runAndReset()) {
            //设置下次任务执行时间
            setNextRunTime();
            //重新排队周期性任务
            reExecutePeriodic(outerTask);
        }
    }
    //设置下次任务执行时间
    private void setNextRunTime() {
        long p = period;
        if (p > 0)
            time += p;
        else
            time = triggerTime(-p);
    }
    //重复执行周期性任务
    void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        //判断线程池状态
        if (canRunInCurrentRunState(true)) {
            //添加任务到队列
            super.getQueue().add(task);
            //再次判断线程池状态，队列移除任务，取消任务
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else//添加线程执行任务
                ensurePrestart();
        }
    }    
}
```
##### 构造方法
```
    ScheduledFutureTask(Callable<V> callable, long ns) {
        super(callable);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
    ScheduledFutureTask(Runnable r, V result, long ns) {
        super(r, result);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
    ScheduledFutureTask(Runnable r, V result, long ns, long period) {
        super(r, result);
        this.time = ns;
        this.period = period;
        this.sequenceNumber = sequencer.getAndIncrement();
    }
```
##### compareTo(Delayed other)
排序算法，首先按照time排序，time小的排在前面，大的排在后面，如果time相同，则使用sequenceNumber排序，小的排在前面，大的排在后面。
```
    public int compareTo(Delayed other) {
        if (other == this) // compare zero if same object
            return 0;
        if (other instanceof ScheduledFutureTask) {
            ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
            long diff = time - x.time;
            if (diff < 0)
                return -1;
            else if (diff > 0)
                return 1;
            else if (sequenceNumber < x.sequenceNumber)
                return -1;
            else
                return 1;
        }
        long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
        return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
    }
```
### 构造方法

```
    //设置核心线程数
    public ScheduledThreadPoolExecutor(int corePoolSize) {
        //调用ThreadPoolExecutor的构造方法
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
    //设置核心线程数和拒绝策略
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), handler);
    }
    //设置核心线程数和线程工厂
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory);
    }
    //设置核心线程数、线程工厂和拒绝策略
    public ScheduledThreadPoolExecutor(int corePoolSize,
                                       ThreadFactory threadFactory,
                                       RejectedExecutionHandler handler) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue(), threadFactory, handler);
    }    
```
### 定时任务
- 未来执行一次的任务，无返回值；
    - `schedule(Runnable command,long delay, TimeUnit unit)`
- 未来执行一次的任务，有返回值；
    - `schedule(Callable<V> callable,long delay, TimeUnit unit)`
- 未来按固定频率重复执行的任务；
    - `scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit)`
- 未来按固定延时重复执行的任务；
    - `scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit)`

#### schedule(Runnable command,long delay, TimeUnit unit)

```
    public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        //参数判断
        if (command == null || unit == null)
            throw new NullPointerException();
        //普通任务包装成ScheduledFutureTask
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        //延时执行
        delayedExecute(t);
        return t;
    }
```
#### schedule(Callable<V> callable,long delay,TimeUnit unit)

```
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
    }
```
#### scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit)

```
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        //参数判断                                                 
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0)
            throw new IllegalArgumentException();
        //普通任务包装成ScheduledFutureTask
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //延时执行
        delayedExecute(t);
        return t;
    }
```
#### scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit)

```
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0)
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        delayedExecute(t);
        return t;
    }
```
#### delayedExecute(RunnableScheduledFuture<?> task)
延时执行任务。
```
    private void delayedExecute(RunnableScheduledFuture<?> task) {
        //线程池关闭了。执行拒绝策略
        if (isShutdown())
            reject(task);
        else {
            //任务添加到队列中
            super.getQueue().add(task);
            //检查线程池状态，如果关闭需要判断关闭策略
            if (isShutdown() &&
                !canRunInCurrentRunState(task.isPeriodic()) &&
                remove(task))//移除任务
                //取消任务
                task.cancel(false);
            else//启动一个新的线程等待执行任务
                ensurePrestart();
        }
    }
    void ensurePrestart() {
        //当前工作线程数
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        //即使corePoolSize为0也会新增线程
        else if (wc == 0)
            addWorker(null, false);
    }
```

![image](https://s2.ax1x.com/2019/12/24/lPQ3CD.png)

### 普通任务
`execute`和`submit`当做普通任务执行。
```
    public void execute(Runnable command) {
        schedule(command, 0, NANOSECONDS);
    }
    public <T> Future<T> submit(Callable<T> task) {
        return schedule(task, 0, NANOSECONDS);
    }
    public Future<?> submit(Runnable task) {
        return schedule(task, 0, NANOSECONDS);
    }
    public <T> Future<T> submit(Runnable task, T result) {
        return schedule(Executors.callable(task, result), 0, NANOSECONDS);
    }    
```
## 示例
```
public class ScheduledThreadPoolExecutorTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(5);
        System.out.println("currentTimeMillis: " + System.currentTimeMillis());

        //执行一个无返回值的任务，5秒后执行，只执行一次
        executor.schedule(()->{
            System.out.println("currentTimeMillis: " + System.currentTimeMillis());
        },5, TimeUnit.SECONDS);
        //执行一个有返回值的任务，5秒后执行，只执行一次
        ScheduledFuture<String> future = executor.schedule(()->{
            System.out.println("currentTimeMillis: " + System.currentTimeMillis());
            return "returnValue";
        },5,TimeUnit.SECONDS);

        System.out.println(future.get() + System.currentTimeMillis());
        //固定频率执行任务，1秒后执行，每2秒执行一次
        executor.scheduleAtFixedRate(()->{
            System.out.println("固定频率: " + System.currentTimeMillis());
        },1,2,TimeUnit.SECONDS);
        //固定延时执行任务，1秒后执行，每2秒执行一次
        executor.scheduleWithFixedDelay(()->{
            System.out.println("固定延时: " + System.currentTimeMillis());
        },1,2,TimeUnit.SECONDS);
    }
}
```

## 总结

- 指定某个时刻执行任务，是通过延时队列的特性来解决的；
- 重复执行，是通过在任务执行后再次把任务加入到队列中来解决的。