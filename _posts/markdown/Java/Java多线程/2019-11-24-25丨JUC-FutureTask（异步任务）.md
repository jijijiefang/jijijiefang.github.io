---
layout:     post
title:      "Java多线程-25丨JUC-FutureTask（异步任务）"
date:       2019-11-24 22:13:05
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# FutureTask
## 简介
>`FutureTask`实现了`Future`，如获取任务执行结果（get）和取消任务（cancel）等。如果任务尚未完成，获取任务执行结果时将会阻塞。一旦执行结束，任务就不能被重启或取消（除非使用runAndReset执行计算）。`FutureTask` 常用来封装 Callable和Runnable，也可以作为一个任务提交到线程池中执行。

### 类图
![image](https://s2.ax1x.com/2019/11/24/MOFKEV.png)

## 原理
### 状态
`FutureTask`内部维护了一个由`volatile`修饰的int型变量`state`，代表当前任务的运行状态，`state`有七种状态：
- **NEW**：初始状态
- **COMPLETING**：正在设置任务结果
- **NORMAL**：任务正常执行完毕
- **EXCEPTIONAL**：异常退出
- **CANCELLED**：任务取消
- **INTERRUPTING**：正在中断运行任务的线程
- **INTERRUPTED**：线程已中断

![状态变化](https://s2.ax1x.com/2019/11/24/MOysAJ.png)

任务的中间状态是一个瞬态，它非常的短暂。**而且任务的中间态并不代表任务正在执行，而是任务已经执行完了，正在设置最终的返回结果**，所以可以这么说：
>**只要state不处于 NEW 状态，就说明任务已经执行完毕**

### 队列
在FutureTask中，队列的实现是一个单向链表，它表示**所有等待任务执行完毕的线程的集合**。<br>
FutureTask中所使用的队列的结构如下：
![image](https://s2.ax1x.com/2019/11/24/MOoBDA.png)

### CAS操作
静态代码块里初始化操作：
```java
    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
```
## 源码
### WaitNode
内部类，组成单向链表
```java
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```
### 属性

```java
    //任务状态
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    //任务
    private Callable<V> callable;
    //任务结果
    private Object outcome; 
    //任务执行者
    private volatile Thread runner;
    //单项链表
    private volatile WaitNode waiters;
```
- 任务本尊：callable
- 任务的执行者：runner
- 任务的结果：outcome
- 获取任务的结果：state + outcome + waiters
- 中断或者取消任务：state + runner + waiters

### 构造函数
- 传入Callable
- 传入Runnable，使用Executors.callable适配成Callable
```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
`Executors.callable`源码：
```java
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    //Runnable转换器
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }    
```
### run()
`FutureTask.run()`源码：
```java
    public void run() {
        //如果任务状态不为NEW或设置任务线程为当前线程不成功返回
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    //执行任务
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    //设置异常
                    setException(ex);
                }
                if (ran)
                    //设置结果
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            //处理中断
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
任务执行成功或被取消，设置结果`set()`:
```java
    protected void set(V v) {
        //state由NEW设置为COMPLETING
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //CAS成功后v设置给outcome，state设置为NORMAL
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```
完成结果设置后，清理阻塞队列`finishCompletion()`：
```java
    private void finishCompletion() {
        //栈顶
        for (WaitNode q; (q = waiters) != null;) {
            //移除等待线程的WaitNode
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                //遍历唤醒栈中所有阻塞线程
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }
        //自定义实现，以实现一些任务执行结束前的额外操作。
        done();
        //清理
        callable = null;
    }
```
处理中断`handlePossibleCancellationInterrupt()`:
```java
    private void handlePossibleCancellationInterrupt(int s) {
        //如果当前的state是INTERRUPTING，原地自旋，直到state状态转换成终止态。
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt
    }
```
### cancel
`FutureTask.cancel()`继承自`Future`接口:
```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        //根据mayInterruptIfRunning的值将state由NEW设置成INTERRUPTING或者CANCELLED
        //只要state不是NEW就返回false
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    //中断当前正在执行任务的线程，最后将state的状态设为INTERRUPTED
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```
`cancel`方法实际完成以下两种状态转换:

- `NEW -> CANCELLED (mayInterruptIfRunning=false)`
- `NEW -> INTERRUPTING -> INTERRUPTED (mayInterruptIfRunning=true)`

### isCancelled()
该方法用于判断任务是否被取消了,如果一个任务在正常执行完成之前被Cancel掉了, 则返回true。
```java
    public boolean isCancelled() {
        //state >= CANCELLED 包含状态: CANCELLED/ INTERRUPTING/INTERRUPTED
        return state >= CANCELLED;
    }
```
### isDone()
只要state状态不是NEW，则任务已经执行完毕了。
```java
    public boolean isDone() {
        return state != NEW;
    }
```
### get()
阻塞获取执行结果，直到获取到或抛异常。<br>
FutureTask中会涉及到两类线程，一类是执行任务的线程，它只有一个，FutureTask的run方法就由该线程来执行；一类是获取任务执行结果的线程，它可以有多个，并发执行get()获取结果。如果任务还没有执行完，则这些线程就需要进入Treiber栈中挂起，直到任务执行结束，或者等待的线程自身被中断。
```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            //等待完成，或者在中断或超时时中止
            s = awaitDone(false, 0L);
        return report(s);
    }
```

`FutureTask.awaitDone()`:

```java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        //循环
        for (;;) {
            //检测获取get的线程是否中断，将WaitNode从栈中移除
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            //如果任务已经进入终止态（s > COMPLETING），返回state状态;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            //如果正在设置结果（s == COMPLETING），让出当前线程的CPU资源继续等待
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            //如果q=null，还没进入栈等待，包装成WaitNode
            else if (q == null)
                q = new WaitNode();
            //如果q不为null，说明当前线程的WaitNode已经被创建出来了
            //如果queued=false，当前线程还没有入栈,进行入栈
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            //设置阻塞                                                   
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            //什么时候会被唤醒呢？
            //任务执行完毕了，在finishCompletion方法中会唤醒所有在Treiber栈中等待的线程
            //等待的线程自身因为被中断等原因而被唤醒。
            else
                LockSupport.park(this);
        }
    }
```
出栈删除WaitNode：
```java
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }
```
根据state状态返回结果或异常`report()`：
```java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
