---
layout:     post
title:      "21丨JUC-ReentrantReadWriteLock（读写锁）"
date:       2019-11-17 17:46:40
author:     "jiefang"
header-style: text
tags:
    - 多线程
    - JUC
    - 锁
---
# JUC-ReentrantReadWriteLock（读写锁）

## 简介
>ReentrantReadWriteLock维护了一对相关的锁：共享锁readLock和独占锁writeLock。共享锁readLock用于读操作，能同时被多个线程获取；独占锁writeLock用于写入操作，只能被一个线程持有。
读写锁的主要特性：
- 公平性：
    - 非公平锁：默认模式
    - 公平锁：
- 可重入性：`ReentrantReadWriteLock`允许读线程和写线程重复获取读锁或写锁。当所有写锁都被释放，不可重入读线程才允许获取锁。此外，一个写入线程可以获取读锁，但是一个读线程不能获取写锁。
- 锁降级：重入性允许从写锁降级到读锁：首先获取写锁，然后获取读锁，然后释放写锁。不过，从一个读锁升级到写锁是不允许的
- Condition支持：Condition只有在写锁中用到，读锁是不支持Condition的。

## 原理
![类图](https://s2.ax1x.com/2019/11/15/Md0yon.png)

- `ReentrantReadWriteLock`实现了`ReadWriteLock`接口。`ReadWriteLock`是一个读写锁的接口，提供了"获取读锁的readLock()函数" 和 "获取写锁的writeLock()函数"。
- `ReentrantReadWriteLock`有三个内部类：自定义同步器`Sync`，读锁`ReadLock`和写锁`WriteLock`。和`ReentrantLock`一样，`Sync`继承自AQS，也包括了两个内部实现公平锁`FairSync`和非公平锁`NonfairSync`。`ReadLock`和`WriteLock`都实现了`Lock`接口，内部也都分别持有`Sync`对象。所有的同步功能都是通过Sync来实现的。

## 源码

### Sync
ReentrantReadWriteLock.Sync源码：
```
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 最多支持65535(1<<16 -1)个写锁（一个独占线程的重入次数）和65535个读锁；低16位表示写锁计数，高16位表示持有读锁的线程数
    static final int SHARED_SHIFT   = 16;
    // 读锁高16位，读锁个数加1，其实是状态值加 2^16
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    // 锁最大数量
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    // 写锁掩码，用于标记低16位
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    //持有读锁线程数
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    //写锁，重入次数
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }    
    //当前线程持有的读锁重入次数
    private transient ThreadLocalHoldCounter readHolds;
    //最近一个获取读锁成功的线程计数器
    private transient HoldCounter cachedHoldCounter;    
    //第一个获取读锁的线程
    private transient Thread firstReader = null;
    //firstReader的持有数
    private transient int firstReaderHoldCount;  
    //构造函数
    Sync() {
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // ensures visibility of readHolds
    }
    // 持有读锁的线程计数器
    static final class HoldCounter {
        int count = 0;
        final long tid = getThreadId(Thread.currentThread());
    }
    // 本地线程计数器
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    //尝试获取读锁
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        for (;;) {
            //判断是被独占锁获取
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            //已获取读锁的数量
            int r = sharedCount(c);
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //读锁
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }    
}
```
### ReadLock
读锁为一个可重入的共享锁，它能够被多个线程同时持有，在没有其他写线程访问时，读锁总是或获取成功。
读锁源码：
```
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;

    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }
    //获取读锁
    public void lock() {
        sync.acquireShared(1);
    }
    //获取读锁响应中断
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    //尝试获取读锁
    public boolean tryLock() {
        return sync.tryReadLock();
    }
    //尝试获取读锁带超时
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }
    //释放读锁
    public void unlock() {
        sync.releaseShared(1);
    }
    //读锁是不支持Condition的
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }
    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
```
#####  lock()
ReentrantReadWriteLock.ReadLock.lock()源码：
```
    //获取读锁
    public void lock() {
        sync.acquireShared(1);
    }
```
`lock`方法调用了同步器`sync`的`acquireShared`,实际调用的是AQS的`acquireShared`，`tryAcquireShared`的子类实现源码是：
```
    protected final int tryAcquireShared(int unused) {
        Thread current = Thread.currentThread();
        int c = getState();
        //持有写锁的线程可以获取读锁，如果获取锁的线程不是current线程；则返回-1
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        //当前持有读锁的线程数量
        int r = sharedCount(c);
        //公平锁和非公平锁判断是否需要阻塞等待
        //判断获取读锁线程是否达到最大值
        //CAS+1成功
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            //第一次获取读锁
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                //重入次数
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                //HoldCounter不存在就返回一个新的默认的
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                //如果rh.count == 0,则把这个rh设置到这个线程的ThreadLocal
                else if (rh.count == 0)
                    readHolds.set(rh);
                //计数加1
                rh.count++;
            }
            return 1;
        }
        return fullTryAcquireShared(current);
    }
    
```
`tryAcquireShared()`的作用是尝试获取“读锁/共享锁”。函数流程如下：

- 如果“写锁”已经被持有，这时候可以继续获取读锁，但如果持有写锁的线程不是当前线程，直接返回-1（表示获取失败）；
- 如果在尝试获取锁时不需要阻塞等待（由公平性决定），并且读锁的共享计数小于最大数量MAX_COUNT，则直接通过CAS函数更新读取锁的共享计数，最后将当前线程获取读锁的次数+1。
- 如果第二步执行失败，则调用`fullTryAcquireShared`尝试获取读锁。
`fullTryAcquireShared`源码：
```
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```
`fullTryAcquireShared`是获取读锁的完整版本，用于处理CAS失败、阻塞等待和重入读问题。相对于`tryAcquireShared`来说，执行流程上都差不多，不同的是，它增加了重试机制和对“持有读锁数的延迟读取”的处理。
##### unlock()
ReentrantReadWriteLock.ReadLock.unlock()源码：
```
    public void unlock() {
        sync.releaseShared(1);
    }
```
AQS中的releaseShared()源码：
```
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
AQS中tryReleaseShared在子类ReentrantReadWriteLock.Sync的实现：
```
    protected final boolean tryReleaseShared(int unused) {
        Thread current = Thread.currentThread();
        //是否是第一个获取读锁的线程
        if (firstReader == current) {
            //更新重入次数
            if (firstReaderHoldCount == 1)
                firstReader = null;
            else
                firstReaderHoldCount--;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                //线程本地变量中的重入次数
                rh = readHolds.get();
            int count = rh.count;
            if (count <= 1) {
                readHolds.remove();
                if (count <= 0)
                    throw unmatchedUnlockException();
            }
            --rh.count;
        }
        for (;;) {
            int c = getState();
            //剩余资源
            int nextc = c - SHARED_UNIT;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
```
### WriteLock
写锁就是一个支持可重入的排他锁。
##### lock()
ReentrantReadWriteLock.WriteLock.lock()源码：
```
    public void lock() {
        sync.acquire(1);
    }
```
调用AQS中的acquire，tryAcquire由ReentrantReadWriteLock.Sync实现：
```
    protected final boolean tryAcquire(int acquires) {
        Thread current = Thread.currentThread();
        //当前锁个数
        int c = getState();
        int w = exclusiveCount(c);
        if (c != 0) {
            //if c != 0 and w == 0说明读锁不为0
            //当前线程不是已获取写锁的线程
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            //最大重入次数
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //重入次数加1
            setState(c + acquires);
            return true;
        }
        //是否需要阻塞，写锁一直是false
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        //设置当前线程独占
        setExclusiveOwnerThread(current);
        return true;
    }
```

##### unlock()
ReentrantReadWriteLock.WriteLock.unlock()源码：
```
    public void unlock() {
        sync.release(1);
    }
    
```
调用AQS中的release(),由子类ReentrantReadWriteLock.Sync实现：
```
    protected final boolean tryRelease(int releases) {
        //判断当前线程是否是独占锁线程
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        //判断重入次数
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
```
