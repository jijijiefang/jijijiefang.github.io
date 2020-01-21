---
layout:     post
title:      "Java多线程-31丨JUC-ForkJoinTask"
date:       2019-12-27 18:00:45
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# ForkJoinTask
## 简介
Java注释
>Abstract base class for tasks that run within a {@link ForkJoinPool}.A {@code ForkJoinTask} is a thread-like entity that is much lighter weight than a normal thread.  Huge numbers of tasks and subtasks may be hosted by a small number of actual threads in a ForkJoinPool, at the price of some usage limitations.

翻译
>在`ForkJoinPool`中运行的任务的抽象基类。`ForkJoinTask`是类似于线程的实体，比普通线程的重量轻得多。`ForkJoinPool`中可能由少量实际线程托管大量任务和子任务，但有一些使用限制。

`ForkJoinTask`是`Fork/Join`任务的抽象定义，支持fork（任务分解）/join（任务结果合并）。

除了`ForkJoinTask`，`Fork/Join`框架还提供了两个它的抽象实现，自定义Fork/Join任务时，一般继承这两个类：

- **RecursiveAction**：表示具有返回结果的ForkJoin任务；
- **RecursiveTask**：表示没有返回结果的ForkJoin任务；

### 类图

![ForkJoinTask](https://s2.ax1x.com/2019/12/27/lVCK3R.png)

## 源码

### 属性
```
    /**任务运行状态**/
    volatile int status; // accessed directly by pool and workers
    //屏蔽掉非完成位
    static final int DONE_MASK   = 0xf0000000;  // mask out non-completion bits
    //表示正常完成,是负值
    static final int NORMAL      = 0xf0000000;  // must be negative
    //表示被取消,负值,且小于NORMAL
    static final int CANCELLED   = 0xc0000000;  // must be < NORMAL
    //异常完成,负值,且小于CANCELLED
    static final int EXCEPTIONAL = 0x80000000;  // must be < CANCELLED
    //用于signal,必须不小于1<<16,默认为1<<16.
    static final int SIGNAL      = 0x00010000;  // must be >= 1 << 16
    //后十六位的task标签
    static final int SMASK       = 0x0000ffff;  // short bits for tags
```
### 任务提交

#### fork()
异步提交任务到队列，返回`ForkJoinTask`。
```
    public final ForkJoinTask<V> fork() {
        Thread t;
        if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
            //任务直接入队
            ((ForkJoinWorkerThread)t).workQueue.push(this);
        else//任务包装入队
            ForkJoinPool.common.externalPush(this);
        return this;
    }   
```
#### invoke()
执行任务，阻塞等待结果返回。
```
    public final V invoke() {
        int s;
        if ((s = doInvoke() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
    //实现调用
    private int doInvoke() {
        int s; Thread t; ForkJoinWorkerThread wt;
        //doExec()结果小于0，直接返回结果；否则等待任务完成。
        return (s = doExec()) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (wt = (ForkJoinWorkerThread)t).pool.
            awaitJoin(wt.workQueue, this, 0L) :
            externalAwaitDone();
    }
    //doExec()会调用子类实现的exec方法完成计算。
    //final修饰,运行ForkJoinTask的核心方法.
    final int doExec() {
        int s; boolean completed;
        //仅未完成的任务会运行,其他情况会忽略.
        if ((s = status) >= 0) {
            try {
                //子类方法实现
                completed = exec();
            } catch (Throwable rex) {
                //设置异常状态
                return setExceptionalCompletion(rex);
            }
            if (completed)
                //设置正常结果状态
                s = setCompletion(NORMAL);
        }
        return s;
    }
    //标记任务结果状态,同时根据情况唤醒等待该task的线程.
    private int setCompletion(int completion) {
        for (int s;;) {
            //如果状态小于0返回
            if ((s = status) < 0)
                return s;
            //状态>=0,CAS更新状态
            if (U.compareAndSwapInt(this, STATUS, s, s | completion)) {
                if ((s >>> 16) != 0)
                    //唤醒其他线程
                    synchronized (this) { notifyAll(); }
                return completion;
            }
        }
    }
    //记录异常并且在符合条件时传播异常行为
    private int setExceptionalCompletion(Throwable ex) {
        //记录异常信息到结果
        int s = recordExceptionalCompletion(ex);
        if ((s & DONE_MASK) == EXCEPTIONAL)
            //status去除非完成态标志位(只保留前4位),等于EXCEPTIONAL.内部传播异常
            internalPropagateException(ex);
        return s;
    }    
```
### 阻塞外部线程
#### externalAwaitDone()
阻塞外部非工作线程，直到完成。
```
    private int externalAwaitDone() {
        int s = ((this instanceof CountedCompleter) ? // try helping
                //当前任务是CountedCompleter，使用外部帮助完成,并将完成状态返回
                 ForkJoinPool.common.externalHelpComplete(
                     (CountedCompleter<?>)this, 0) :
                 //当前任务不是CountedCompleter,则调用common pool尝试外部弹出该任务并进行执行,
                //status赋值doExec函数的结果,若弹出失败(其他线程先行弹出)赋0.
                 ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
        //status大于0表示尝试帮助完成失败
        if (s >= 0 && (s = status) >= 0) {
            //中断标识
            boolean interrupted = false;
            do {
                //先给status标记SIGNAL标识,便于后续唤醒操作
                if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                    synchronized (this) {
                        if (status >= 0) {
                            try {
                                //CAS成功，阻塞
                                wait(0L);
                            } catch (InterruptedException ie) {
                                //设置中断标识为true
                                interrupted = true;
                            }
                        }
                        else//唤醒所有阻塞线程
                            notifyAll();
                    }
                }
            } while ((s = status) >= 0);
            //处理线程中断
            if (interrupted)
                Thread.currentThread().interrupt();
        }
        return s;
    }
```
#### externalInterruptibleAwaitDone()
逻辑与`externalAwaitDone()`类似，只是线程中断处理为抛异常。
```
    private int externalInterruptibleAwaitDone() throws InterruptedException {
        int s;
        //判断线程中断
        if (Thread.interrupted())
            throw new InterruptedException();
        if ((s = status) >= 0 &&
            (s = ((this instanceof CountedCompleter) ?
                  ForkJoinPool.common.externalHelpComplete(
                      (CountedCompleter<?>)this, 0) :
                  ForkJoinPool.common.tryExternalUnpush(this) ? doExec() :
                  0)) >= 0) {
            while ((s = status) >= 0) {
                if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                    synchronized (this) {
                        if (status >= 0)
                            wait(0L);
                        else
                            notifyAll();
                    }
                }
            }
        }
        return s;
    }
```
### 返回结果
#### join()
等待任务完成并获取结果，尝试在当前线程中开始执行。
```
    public final V join() {
        int s;
        //如果执行结果不为正常，抛异常
        if ((s = doJoin() & DONE_MASK) != NORMAL)
            reportException(s);
        //返回结果，子类实现
        return getRawResult();
    }
    //join的核心方法
    private int doJoin() {
        int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
        //已完成,返回status,未完成再尝试后续
        return (s = status) < 0 ? s :
            //当前线程是ForkJoinWorkerThread，从该线程中取出workQueue，尝试将
            //当前task出队并执行，执行结果完成则返回状态；否则使用线程所在pool的awaitJoin方法等待
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (w = (wt = (ForkJoinWorkerThread)t).workQueue).
            tryUnpush(this) && (s = doExec()) < 0 ? s :
            wt.pool.awaitJoin(w, this, 0L) :
            //当前线程不是ForkJoinWorkerThread,调用externalAwaitDone方法
            externalAwaitDone();
    }    
```
#### invoke()
提交任务等待任务完成并获取结果（当前线程执行）。
```
    public final V invoke() {
        int s;
        if ((s = doInvoke() & DONE_MASK) != NORMAL)
            reportException(s);
        return getRawResult();
    }
    private int doInvoke() {
        int s; Thread t; ForkJoinWorkerThread wt;
        //直接执行任务，如果结果完成直接返回;
        //判断当前线程是否是ForkJoinWorkerThread，若是调用pool的awaitJoin方法等待;
        //若不是，调用externalAwaitDone方法
        return (s = doExec()) < 0 ? s :
            ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
            (wt = (ForkJoinWorkerThread)t).pool.
            awaitJoin(wt.workQueue, this, 0L) :
            externalAwaitDone();
    }    
```
#### get()
阻塞获取结果。
```
    public final V get() throws InterruptedException, ExecutionException {
        //如果当前是ForkJoinWorkerThread线程直接调用doJoin
        int s = (Thread.currentThread() instanceof ForkJoinWorkerThread) ?
            doJoin() : externalInterruptibleAwaitDone();
        Throwable ex;
        //处理任务取消
        if ((s &= DONE_MASK) == CANCELLED)
            throw new CancellationException();
        //处理任务异常
        if (s == EXCEPTIONAL && (ex = getThrowableException()) != null)
            throw new ExecutionException(ex);
        return getRawResult();
    }
```
### 辅助方法
```
    //取消一个任务的执行,直接将status设置成CANCELLED,设置后判断该status 是否为CANCELLED,是则true否则false.
    public boolean cancel(boolean mayInterruptIfRunning) {
        return (setCompletion(CANCELLED) & DONE_MASK) == CANCELLED;
    }
    //判断是否完成,status小于0代表正常完成/异常完成/取消,
    public final boolean isDone() {
        return status < 0;
    }
    //判断当前任务是否取消.
    public final boolean isCancelled() {
        return (status & DONE_MASK) == CANCELLED;
    }
    //否为异常完成,CANCELLED和EXCEPTIONAL均小于NORMAL
    public final boolean isCompletedAbnormally() {
        return status < NORMAL;
    }
    //是否正常完成.
    public final boolean isCompletedNormally() {
        return (status & DONE_MASK) == NORMAL;
    }
    //获取异常
    public final Throwable getException() {
        int s = status & DONE_MASK;
        return ((s >= NORMAL)    ? null :
                (s == CANCELLED) ? new CancellationException() :
                getThrowableException());
    }
    //使用异常完成任务
    public void completeExceptionally(Throwable ex) {
        setExceptionalCompletion((ex instanceof RuntimeException) ||
                                 (ex instanceof Error) ? ex :
                                 new RuntimeException(ex));
    }
    //使用value完成任务
    public void complete(V value) {
        try {
            //设置原始结果
            setRawResult(value);
        } catch (Throwable rex) {
            //异常方式完成
            setExceptionalCompletion(rex);
            return;
        }
        //标记完成
        setCompletion(NORMAL);
    }
    //安静完成任务
    public final void quietlyComplete() {
        setCompletion(NORMAL);
    }
    //安静join,它不会返回result也不会抛出异常
    public final void quietlyJoin() {
        doJoin();
    }
    //安静执行一次,不返回结果不抛出异常
    public final void quietlyInvoke() {
        doInvoke();
    }
    //重新初台化当前task
    public void reinitialize() {
        if ((status & DONE_MASK) == EXCEPTIONAL)
            clearExceptionalCompletion();
        else
            status = 0;
    }
    //反fork
    public boolean tryUnfork() {
        Thread t;
        return (((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
                ((ForkJoinWorkerThread)t).workQueue.tryUnpush(this) :
                ForkJoinPool.common.tryExternalUnpush(this));
    }    
```
## 子类
### RecursiveAction
适合only fork：递归划分子任务，分别执行，但是并不需要合并各自的执行结果。

![image](https://s2.ax1x.com/2019/12/27/lZCAeK.png)

```
public abstract class RecursiveAction extends ForkJoinTask<Void> {
    private static final long serialVersionUID = 5232453952276485070L;
    //任务计算
    protected abstract void compute();
    //无返回值
    public final Void getRawResult() { return null; }
    protected final void setRawResult(Void mustBeNull) { }
    //
    protected final boolean exec() {
        compute();
        return true;
    }

}
```
### RecursiveTask
适合fork+join：递归划分子任务，分别执行，然后递归合并计算结果。

![image](https://s2.ax1x.com/2019/12/27/lZCQyt.png)

```
public abstract class RecursiveTask<V> extends ForkJoinTask<V> {
    private static final long serialVersionUID = 5232453952276485270L;
    //结果
    V result;
    //任务计算
    protected abstract V compute();

    public final V getRawResult() {
        return result;
    }

    protected final void setRawResult(V value) {
        result = value;
    }

    protected final boolean exec() {
        result = compute();
        return true;
    }

}
```
## 总结
- 任务状态
    - 当状态完成时，为负数，表示正常完成、取消或者异常；
    - 阻塞等待的任务设置了SIGNAL；
- 内部提交
    - fork:直接加入到当前线程的workQueue中;
    - invoke:提交任务等待任务完成并获取结果（当前线程执行）;
    - join:等待任务完成并获取结果，尝试在当前线程中开始执行;
- 任务类型
    - `RecursiveAction`:不返回结果的计算;
    - `RecursiveTask`:返回结果;