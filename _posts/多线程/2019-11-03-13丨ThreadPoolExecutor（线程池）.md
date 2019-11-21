---
layout:     post
title:      "13丨ThreadPoolExecutor（线程池）"
date:       2019-11-03 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
    - JUC
---
# 线程池

## 概述
>ThreadPoolExecutor 是线程池的核心实现。线程的创建和终止需要很大的开销，线程池中预先提供了指定数量的可重用线程，所以使用线程池会节省系统资源，并且每个线程池都维护了一些基础的数据统计，方便线程的管理和监控。


## 参数
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

参数名 | 作用
---|---
corePoolSize| 核心线程池大小
maximumPoolSize| 最大线程池大小
keepAliveTime| 线程池中超过corePoolSize数目的空闲线程最大存活时间；可以allowCoreThreadTimeOut(true)使得核心线程空闲存活有效时间
TimeUnit |keepAliveTime时间单位
workQueue |阻塞任务队列
threadFactory |线程工厂
RejectedExecutionHandler| 当提交任务数超过maxmumPoolSize+workQueue之和时，任务会交给RejectedExecutionHandler来处理；

## 流程
![image](https://s2.ax1x.com/2019/11/02/KOSEPU.png)
- 当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。
- 当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行
- 当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务
- 当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理
- 当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程
- 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭

## 拒绝策略
- **AbortPolicy**:默认策略，在需要拒绝任务时抛出RejectedExecutionException；
- **DiscardPolicy**:也是丢弃任务，但是不抛出异常。 
- **DiscardOldestPolicy**:丢弃队列中等待时间最长的任务，并执行当前提交的任务，如果线程池已经关闭，任务将被丢弃。
- **CallerRunsPolicy**:直接在 execute 方法的调用线程中运行被拒绝的任务，如果线程池已经关闭，任务将被丢弃；

## 线程池状态
- **RUNNING**:可以接收新的任务和队列任务
- **SHUTDOWN**:不接收新的任务，但是会运行队列任务
- **STOP**:不接收新任务，也不会运行队列任务，并且中断正在运行的任务
- **TIDYING**:所有任务都已经终止，workerCount为0，当池状态为TIDYING时将会运行terminated()方法
- **TERMINATED**:terminated函数完成执行

![image](https://s2.ax1x.com/2019/11/02/KO9hCT.md.png)

## UML图

![UML图](https://s2.ax1x.com/2019/11/21/M5LjoD.png)
## 源码
### Worker
ThreadPoolExecutor.Worker源码：
```
//ThreadPoolExecutor 的内部类，继承自 AQS，实现了不可重入的互斥锁。
//在线程池中持有一个 Worker 集合，一个 Worker 对应一个工作线程。
//当线程池启动时，对应的Worker会执行池中的任务，执行完毕后从阻塞队列里获取一个新的任务继续执行。
//它本身实现了Runnable接口，也就是说 Worker本身也作为一个线程任务执行。
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable{
        private static final long serialVersionUID = 6138294804551838833L;
        //工作线程
        final Thread thread;
        //初始运行任务
        Runnable firstTask;
        //任务完成计数
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        
        public void run() {
            runWorker(this);
        }
        //0代表解锁状态;1代表锁定状态
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
        //实现AQS方法，尝试获取锁，CAS成功设置独占，返回true
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
        //实现AQS方法，尝试释放锁
        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
        //获取锁
        public void lock()        { acquire(1); }
        //尝试获取锁
        public boolean tryLock()  { return tryAcquire(1); }
        //释放锁
        public void unlock()      { release(1); }
        //是否锁定
        public boolean isLocked() { return isHeldExclusively(); }
        //锁定后中断线程
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
### ThreadPoolExecutor
ThreadPoolExecutor核心属性：
```
public class ThreadPoolExecutor extends AbstractExecutorService {
    //ctl封装了两个字段：workerCount(有效线程数)和runState(线程池状态)
    //ctl使用低29位表示线程池中的线程数，高3位表示线程池的运行状态
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    //任务线程数量所占的int的位数
    private static final int COUNT_BITS = Integer.SIZE - 3;
    //最大任务线程数量为2^29-1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    //对应高三位111
    private static final int RUNNING    = -1 << COUNT_BITS;
    //对应高三位000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    //对应高三位001
    private static final int STOP       =  1 << COUNT_BITS;
    //对应高三位010
    private static final int TIDYING    =  2 << COUNT_BITS;
    //对应高三位011
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    //运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    //运行的任务线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    //封装运行状态和任务线程
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    //当核心线程数已满，新增任务的存储队列
    private final BlockingQueue<Runnable> workQueue;
    //线程运行期间的锁，在调用shutdown和shutdownNow之后依然持有
    private final ReentrantLock mainLock = new ReentrantLock();
    //工作线程池，只有在持有mainLock才存储
    private final HashSet<Worker> workers = new HashSet<Worker>();
    //awaitTermination的等待条件
    private final Condition termination = mainLock.newCondition();
    //当前线程池里线程数量
    private int largestPoolSize;
    //已完成任务数量
    private long completedTaskCount;
    //线程工厂，所有线程都是用它来创建
    private volatile ThreadFactory threadFactory;
    //拒绝策略
    private volatile RejectedExecutionHandler handler;
    //空闲线程存活时间
    private volatile long keepAliveTime;
    //默认false，表示core线程空闲依然保活；如果为true，使用keepAliveTime确定等待超时时间
    private volatile boolean allowCoreThreadTimeOut;
    //核心线程池大小
    private volatile int corePoolSize;
    //最大线程池大小
    private volatile int maximumPoolSize;
    //默认拒绝策略
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
    //针对shutdown和shutdownNow的运行权限许可
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");
}
```
#### execute()
ThreadPoolExecutor.execute()方法源码：
```
    //提交一个任务到线程池，任务不一定会立即执行。
    //提交的任务可能在一个新的线程中执行，也可能在已经存在的空闲线程中执行。
    //如果由于池关闭或者池容量已经饱和导致任务无法提交，那么就根据拒绝策略RejectedExecutionHandler处理提交过来的任务。
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```
`execute`运行的三种情况：

- 如果正在运行线程少于corePoolSize，通过addWorker方法尝试开启一个新的线程并把提交的任务作为它的firstTask运行。addWorker会检查ctl状态的状态（runState和workerCount）来判断是否可以添加新的线程。
- 如果addWorker执行失败（返回false），就把任务添加到等待队列。这里需要对ctl进行双重检查，因为从任务入队到入队完成可能有线程死掉，或者在进入此方法后线程池被关闭。所以我们要在入队后重新检查池状态，如果有必要，就回滚入队操作。
- 如果任务不能入队，我们再次尝试增加一个新的线程。如果添加失败，就意味着池被关闭或已经饱和，这种情况就需要根据拒绝策略来处理任务。

#### addWorker()
ThreadPoolExecutor.addWorker()源码：
```
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //检查线程池状态
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                //工作线程数
                int wc = workerCountOf(c);
                //工作线程数是否大于线程池最大容量或是否大于核心线程数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                //工作线程数+1
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                //如果第一次获取的工作线程数和当前获取的不相同跳出
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            //创建新的工作线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                //加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //检查线程池状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        //线程已经运行则抛异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        //更新largestPoolSize
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                //成功加入，启动线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            //启动失败，回滚
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

#### addWorkerFailed()
ThreadPoolExecutor.addWorkerFailed()源码：
```
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            //尝试终止线程池
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
```
#### runWorker
`ThreadPoolExecutor.addWorker()`方法里调用`t.start()`时调用runWorker()。
```
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        //Worker初始化时设置state为-1，这里设置为0，允许中断，在任务未执行前不允许中断
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            //task不等于null或者阻塞队列中getTask()不等于null
            while (task != null || (task = getTask()) != null) {
                //加锁
                w.lock();
                //线程池调用shutdownNow()方法后状态变为STOP
                //线程池当前状态为STOP或者当前线程中断且线程池状态为STOP
                //且线程还没有中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    //中断Worker线程
                    wt.interrupt();
                try {
                    //执行前逻辑，自定义实现
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //任务运行
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //执行后逻辑，自定义实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //处理工作线程退出逻辑
            processWorkerExit(w, completedAbruptly);
        }
    }
```
#### getTask
ThreadPoolExecutor.getTask()源码：
```
    //从阻塞队列中取任务的方法
    private Runnable getTask() {
        //队列中取任务是否超时
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //线程池状态>=STOP
            //或线程池状态>=SHUTDOWN且队列为空
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                //减少ctl的workerCount数
                decrementWorkerCount();
                return null;
            }
            //工作线程数
            int wc = workerCountOf(c);

            //核心线程空闲超时设置或当前线程数大于核心线程数
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            //1、当前线程数大于最大线程数
            //2、获取任务超时且(核心线程设置空闲超时或当前线程大于最大线程)
            //3、任务队列为空
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)://获取任务等待指定时间
                    workQueue.take();//获取任务阻塞直到获取到
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

ThreadPoolExecutor.getTask()以下情况会返回null，使Work线程退出：
- 工作线程数大于`maximumPoolSize`;
- 线程池已停止（STOP）
- 线程池已关闭（SHUTDOWN）且阻塞队列为空
- 等待任务超时

#### processWorkerExit
`runWorker()`中线程最后处理完任务后，调用`processWorkerExit`退出逻辑:
```
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        //如果线程中断，会抛异常，进入finally,则completedAbruptly为true
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //更新完成任务数
            completedTaskCount += w.completedTasks;
            //移除Worker
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
        //尝试终止线程池
        tryTerminate();

        int c = ctl.get();
        //线程池处于RUNNING或SHUTDOWN状态，并未完全停止
        if (runStateLessThan(c, STOP)) {
            //工作线程非异常退出
            if (!completedAbruptly) {
                //获取核心线程数
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                //是否允许核心线程空闲且任务队列不为空，则min为1
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                //当前工作线程数>=min返回
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            //添加一个没有firstTask的worker
            addWorker(null, false);
        }
    }
```
#### tryTerminate
尝试终止线程池,在shutdow()、shutdownNow()、remove()中均是通过此方法来终止线程池。
```
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            //正在运行
            //处于STOP状态
            //处于SHUTDOWN状态且阻塞队列不为空
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            //此处时说明线程池处于STOP状态，或者处于处于SHUTDOWN状态且阻塞队列为空
            //如果线程池中还存在线程，则会尝试中断线程
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
            
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                //线程池已经关闭，等待队列为空，并且工作线程等于0，更新池状态为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        //线程池关闭以后方法，需自定义实现
                        terminated();
                    } finally {
                        //最后更新线程池状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        //唤醒等待池结束的线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

#### shutdown
启动一个有序的关闭方式，在关闭之前已提交的任务会被执行，但不会接收新任务。
```
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查关闭权限
            checkShutdownAccess();
            //修改线程池状态SHUTDOWN
            advanceRunState(SHUTDOWN);
            //中断空闲工作线程
            interruptIdleWorkers();
            //关闭线程池调用，需自定义实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        //尝试终止线程池
        tryTerminate();
    }
```
#### shutdownNow
停止线程池内所有的任务（包括正在执行和正在等待的任务），并返回正在等待执行的任务列表。
```
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            //检查关闭权限
            checkShutdownAccess();
            //修改线程池状态STOP
            advanceRunState(STOP);
            //中断所有线程
            interruptWorkers();
            //移除阻塞队列中的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        //尝试终止线程池
        tryTerminate();
        return tasks;
    }
```
shutdcown 和 shutdownNow的区别：
- `shutdown` 会把当前池状态改为`SHUTDOWN`，表示还会继续运行池内已经提交的任务，然后中断所有的空闲工作线程 ；但 `shutdownNow` 直接把池状态改为`STOP`，也就是说不会再运行已存在的任务，然后会**中断所有工作线程**。