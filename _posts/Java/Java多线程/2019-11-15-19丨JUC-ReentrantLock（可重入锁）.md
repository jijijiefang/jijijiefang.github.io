---
layout:     post
title:      "Java多线程-19丨JUC-ReentrantLock（可重入锁）"
date:       2019-11-14 18:16:06
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
    - 锁
---
# ReentrantLock
## 简介
官方定义
>可重入的互斥锁，其基本行为和语义与使用synchronized方法和语句访问的隐式监视器锁相同，但具有扩展的功能。<br>
`可重入`的意思是可以被线程多次获取锁。<br>`ReentrantLock`又分为**公平锁**(fair lock)和**非公平锁**(non-fair lock)。它们的区别体现在获取锁的机制上：在“公平锁”的机制下，线程依次排队获取锁；而“非公平锁”机制下，如果锁是可获取状态，不管自己是不是在队列的head节点都会去尝试获取锁。

## 原理
### UML图
![image](https://s2.ax1x.com/2019/11/15/MdCKPg.png)
### 方法
`ReentrantLock`继承自接口`Lock`,并持有内部类`Sync`、`NonfairSync`、`FairSync`。

`Lock`中的方法：

```java
public interface Lock {
    //获取锁，如果锁不可用则线程一直等待
    void lock();
    //获取锁，响应中断，如果锁不可用则线程一直等待
    void lockInterruptibly() throws InterruptedException;
    //获取锁，获取失败直接返回
    boolean tryLock();
    //获取锁，等待给定时间后如果获取失败直接返回
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    //释放锁
    void unlock();
    //创建一个新的等待条件
    Condition newCondition();
}
```
`ReentrantLock`对`Lock`的实现：
`ReentrantLock`对于`Lock`接口的实现都是直接“转交”给其内部类的*sync*对象的。

Lock 接口 | ReentrantLock 实现
---|---
`lock()`	|`sync.lock()`
`lockInterruptibly()`	|`sync.acquireInterruptibly(1)`
`tryLock()`	|`sync.nonfairTryAcquire(1)`
`tryLock(long time, TimeUnit unit)`	|`sync.tryAcquireNanos(1, unit.toNanos(timeout))`
`unlock()`	|`sync.release(1)`
`newCondition()`	|`sync.newCondition()`

## 源码
### 构造函数
`ReentrantLock`的构造函数有两个实现，分别是公平锁（默认）和非公平锁：

```java
    //非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    //根据传入变量确定
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
为什么要区分公平锁和非公平锁？
>主要原因是：在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟。<br>
在公平锁模式下，大家讲究先来后到，如果当前线程A在请求锁，即使现在锁处于可用状态，它也得在队列的末尾排着，这时我们需要唤醒排在等待队列队首的线程H(在AQS中其实是次头节点)，由于恢复一个被挂起的线程并且让它真正运行起来需要较长时间，那么这段时间锁就处于空闲状态，时间和资源就白白浪费;<br>
非公平锁的设计思想就是将这段白白浪费的时间利用起来——由于线程A在请求锁的时候本身就处于运行状态，因此如果我们此时把锁给它，它就会立即执行自己的任务，因此线程A有机会在线程H完全唤醒之前获得、使用以及释放锁。这样我们就可以把线程H恢复运行的这段时间给利用起来了，结果就是线程A更早的获取了锁，线程H获取锁的时刻也没有推迟。因此提高了吞吐量。<br>
当然，非公平锁仅仅是在当前线程请求锁，并且锁处于可用状态时有效，当请求锁时，锁已经被其他线程占有时，就只能还是老老实实的去排队了。
### lock()
`ReentrantLock.NonfairSync.lock()`：

```java
    final void lock() {
        //尝试使用CAS操作去抢锁
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
```
`ReentrantLock.FairSync.lock()：`

```java
    final void lock() {
        acquire(1);
    }
```
##### tryAcquire()
AQS中的`tryAcquire()`方法由子类实现：
`ReentrantLock.NonfairSync.tryAcquire()`：

```java
    //调用Sync中的nonfairTryAcquire()
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
    //Sync中的nonfairTryAcquire方法
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }    
```
`ReentrantLock.FairSync.tryAcquire()`：

```java
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            //是否有线程排在自己前面
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            //重入锁
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```
唯一的区别就是非公平锁在抢锁时不再需要调用`hasQueuedPredecessors`方法先去判断是否有线程排在自己前面，而是直接争锁，其它的完全和公平锁一致。
### unlock()
```java
    //释放锁
    public void unlock() {
        sync.release(1);
    }
```
##### tryRelease()
AQS中`release()`方法由子类实现：
`ReentrantLock.Sync.tryRelease()`:

```java
    //在可重入锁中，获取锁的次数必须要等于释放锁的次数，这样才算是真正释放了锁
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //锁完全释放成功
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```
### tryLock()
`tryLock()`仅仅是用于检查锁在当前调用的时候是不是可获得的，所以即使现在使用的是非公平锁，在调用这个方法时，当前线程也会直接尝试去获取锁，哪怕这个时候队列中还有在等待中的线程。所以这一方法对于公平锁和非公平锁的实现是一样的，它被定义在`Sync`类中，由`FairSync`和`NonfairSync`直接继承使用。

```java
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }
```
## ReentrantLock与synchronized
`ReentrantLock`与`synchronized`有着相同的功能和内存语义，但是还是有一些差异：

- 与`synchronized`相比，`ReentrantLock`提供了更多，更加全面的功能，具备更强的扩展性。例如：**时间锁等候，可中断锁等候，公平性，以及实现非块结构的加锁**。
- `ReentrantLock`还提供了条件`Condition`，对线程的等待、唤醒操作更加详细和灵活，所以在**多个条件变量**的地方，`ReentrantLock`更加适合。
- `ReentrantLock`提供了`tryLock()`的锁请求。它会**尝试着去获取锁**，如果成功则继续，否则可以等到下次运行时处理，而`synchronized`则一旦进入锁请求要么成功要么阻塞，所以相比`synchronized`而言，`ReentrantLock`会不容易产生死锁些。
- `ReentrantLock`支持**更加灵活的同步代码块**，但是使用`synchronized`时，只能在同一个`synchronized`块结构中获取和释放。注：`ReentrantLock`的锁释放一定要在finally中处理，否则可能会产生严重的后果。
- `ReentrantLock`**支持中断处理**。
- JDK5的早期版本中，`ReentrantLock`的性能远远好于`synchronized`，但是从JDK6开始，JDK在`synchronized`上做了大量优化，使得两者的性能差距不大。


功能 | ReentrantLock|synchronized
---|---|---
可重入|支持|支持
非公平|支持（默认）|支持
加锁/解锁方式|需要手动加锁、解锁，一般使用try..finally..保证锁能够释放|手动加锁，无需刻意解锁
按key锁|不支持，比如按用户id加锁	|支持，synchronized加锁时需要传入一个对象
公平锁|支持，new ReentrantLock(true)|不支持
中断|支持，lockInterruptibly()|不支持
尝试加锁|支持，tryLock()|不支持
超时锁|支持，tryLock(timeout, unit)|不支持
获取当前线程获取锁的次数|支持，getHoldCount()|不支持
获取等待的线程|支持，getWaitingThreads()|不支持
检测是否被当前线程占有|支持，isHeldByCurrentThread()|不支持
检测是否被任意线程占有|支持，isLocked()|不支持
条件锁|可支持多个条件，condition.await()，condition.signal()，condition.signalAll()|只支持一个，obj.wait()，obj.notify()，obj.notifyAll()

