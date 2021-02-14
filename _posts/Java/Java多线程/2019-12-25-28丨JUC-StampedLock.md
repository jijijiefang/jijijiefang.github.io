---
layout:     post
title:      "Java多线程-28丨JUC-StampedLock"
date:       2019-12-25 22:45:22
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# StampedLock
`StampedLock`类，在JDK1.8时引入，是对读写锁`ReentrantReadWriteLock`的增强，该类提供了一些功能，优化了读锁、写锁的访问，同时使读写锁之间可以互相转换，更细粒度控制并发。
## 简介
Java注释
>A capability-based lock with three modes for controlling read/write access.  The state of a StampedLock consists of a version and mode.Lock acquisition methods return a stamp that represents and controls access with respect to a lock state; "try" versions of these methods may instead return the special value zero to represent failure to acquire access. Lock release and conversion methods require stamps as arguments, and fail if they do not match the state of the lock.

翻译
>基于功能的锁，具有三种模式来控制读/写访问。 `StampedLock`的状态由版本和模式组成。 锁定获取方法返回一个表示并控制相对于锁定状态的访问的标记；这些方法的“尝试”版本可能会改为将特殊值零返回给，表示无法获取访问权限。锁释放和转换方法需要使用图章作为参数，并且如果它们与锁的状态不匹配，则会失败。

`StampedLock`具有三种模式：写模式、读模式、乐观读模式。
### 类图

![StampedLock](https://s2.ax1x.com/2019/12/24/lPrkcR.png)

## 原理
### StampedLock的引入
>为什么有了`ReentrantReadWriteLock`，还要引入`StampedLock`？

`ReentrantReadWriteLock`使得多个读线程同时持有读锁（只要写锁未被占用），而写锁是独占的。但是，读写锁如果使用不当，很容易产生“饥饿”问题：

比如在读线程非常多，写线程很少的情况下，很容易导致写线程“饥饿”，虽然使用“公平”策略可以一定程度上缓解这个问题，但是“公平”策略是以牺牲系统吞吐量为代价的。

### 特点
1. 所有获取锁的方法，都返回一个邮戳（Stamp），Stamp为0表示获取失败，其余都表示成功；
2. 所有释放锁的方法，都需要一个邮戳（Stamp），这个Stamp必须是和成功获取锁时得到的Stamp一致；
3. `StampedLock`是不可重入的；（如果一个线程已经持有了写锁，再去获取写锁的话就会造成死锁）；
4. `StampedLock`有三种访问模式：
    - **Reading（读模式）**：功能和`ReentrantReadWriteLock`的读锁类似；
    - **Writing（写模式）**：功能和`ReentrantReadWriteLock`的写锁类似；
    - **Optimistic reading（乐观读模式）**：这是一种优化的读模式；
5. `StampedLock`支持读锁和写锁的相互转换；`ReentrantReadWriteLock`中，当线程获取到写锁后，可以降级为读锁，但是读锁是不能直接升级为写锁的；`StampedLock`提供了读锁和写锁相互转换的功能；
6. 无论写锁还是读锁，都不支持`Conditon`等待；

### 数据结构

![image](https://s2.ax1x.com/2019/12/25/lFvIyQ.png)

## 示例
```java
public class StampedLockTest {
    public static void main(String[] args) {
        int x = 0;
        int y = 0;
        StampedLock stampedLock = new StampedLock();
        //获取写锁，返回版本号
        long stamp = stampedLock.writeLock();
        try {
            x += 1;
            y += 1;
        }finally {
            //释放写锁，传入上面的版本号
            stampedLock.unlockWrite(stamp);
        }
        //乐观读
        stamp = stampedLock.tryOptimisticRead();
        //验证版本号是否有变化
        if(!stampedLock.validate(stamp)){
            //版本号变化后，由乐观读变悲观读
            stamp = stampedLock.readLock();

            try {
                x += 1;
                y += 1;
            }finally {
                //释放读锁，传入上面的版本号
                stampedLock.unlockRead(stamp);
            }
        }
        //获取悲观读锁
        stamp = stampedLock.readLock();

        try {
            while (x == 0 && y == 0){
                //读锁尝试转化为写锁
                long next = stampedLock.tryConvertToWriteLock(stamp);
                //转化成功
                if(next != 0L){
                    stamp = next;
                    x +=1;
                    y +=1;
                //转化失败
                }else {
                    //释放读锁
                    stampedLock.unlockRead(stamp);
                    //获取写锁
                    stamp  = stampedLock.writeLock();
                }
            }
        }finally {
            //释放写锁
            stampedLock.unlockWrite(stamp);
        }
    }
}
```
## 源码

### 属性
```java
    //CPU核数
    private static final int NCPU = Runtime.getRuntime().availableProcessors();
    //入队前最大重试次数
    private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;
    //获取锁前最大重试次数
    private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;
    //再次阻塞前最大重试次数
    private static final int MAX_HEAD_SPINS = (NCPU > 1) ? 1 << 16 : 0;
    //自旋锁超时yielding次数
    private static final int OVERFLOW_YIELD_RATE = 7; // must be power 2 - 1
    //读线程的个数占有低7位
    private static final int LG_READERS = 7;
    //锁定状态和标记操作的值
    //读线程个数每次增加的单位
    private static final long RUNIT = 1L;
    //写线程个数所在的位置
    private static final long WBIT  = 1L << LG_READERS;
    //读线程个数所在的位置
    private static final long RBITS = WBIT - 1L;
    //最大读线程个数
    private static final long RFULL = RBITS - 1L;
    //读线程个数和写线程个数的掩码
    private static final long ABITS = RBITS | WBIT;
    //读线程个数的反数，高25位全部为1
    private static final long SBITS = ~RBITS; // note overlap with ABITS
    //state的初始值
    private static final long ORIGIN = WBIT << 1;
    //取消获取方法中的特殊值，因此调用方可以抛出IE
    private static final long INTERRUPTED = 1L;
    //节点状态
    private static final int WAITING   = -1;
    private static final int CANCELLED =  1;
    //节点模式0：读模式；1：写模式
    private static final int RMODE = 0;
    private static final int WMODE = 1;
    
    //队列头结点
    private transient volatile WNode whead;
    //队列尾节点
    private transient volatile WNode wtail;

    transient ReadLockView readLockView;
    transient WriteLockView writeLockView;
    transient ReadWriteLockView readWriteLockView;
    //存储着当前的版本号，类似于AQS的状态变量state
    private transient volatile long state;
    //
    private transient int readerOverflow;    
```

![image](https://s2.ax1x.com/2019/12/25/lFxspT.md.png)

### 内部类
#### WNode
类似AQS队列中的节点，组成双向链表，内部维护着阻塞的线程。
```java
    static final class WNode {
        //前置节点
        volatile WNode prev;
        //后继节点
        volatile WNode next;
        //读线程所用的链表（实际是一个栈结果）
        volatile WNode cowait;    // list of linked readers
        //阻塞的线程
        volatile Thread thread;   // non-null while possibly parked
        //状态
        volatile int status;      // 0, WAITING, or CANCELLED
        //读模式还是写模式
        final int mode;           // RMODE or WMODE
        WNode(int m, WNode p) { mode = m; prev = p; }
    }
```
### 构造方法
state的初始值为ORIGIN（256），它的二进制是 1 0000 0000，也就是初始版本号。
```java
    public StampedLock() {
        state = ORIGIN;
    }
```
### writeLock()
获取写锁。
```java
    //以独占方式获取锁，必要时阻塞，直到可用为止
    public long writeLock() {
        long s, next;  // bypass acquireWrite in fully unlocked case only
        //WBIT  = 1L << 7 = 128 = 1000 0000;
        //ABITS = 255 = 1111 1111
        //state = 1 0000 0000
        //state与ABITS如果等于0，尝试CAS更新state的值加WBITS
        //成功则返回更新的值，如果失败调用acquireWrite()方法
        //CAS更新成功后，next = 384 = 1 1000 0000
        return ((((s = state) & ABITS) == 0L &&
                 U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
                next : acquireWrite(false, 0L));
    }
```
![image](https://s2.ax1x.com/2019/12/25/lFEzx1.png)

#### acquireWrite(boolean interruptible, long deadline)

```java
    private long acquireWrite(boolean interruptible, long deadline) {
        WNode node = null, p;
        //入队自旋
        for (int spins = -1;;) { // spin while enqueuing
            long m, s, ns;
            //如果state & ABITS等于0，CAS更新，获取写锁
            if ((m = (s = state) & ABITS) == 0L) {
                if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
                    return ns;
            }
            else if (spins < 0)
                //自旋次数小于0，计算自旋次数
                //当前有写锁独占且队列无元素，说明快轮到自己了
                //则自旋次数等于64，否则为0
                spins = (m == WBIT && wtail == whead) ? SPINS : 0;
            else if (spins > 0) {
                //自旋次数大于0时，当前这次自旋随机减一
                if (LockSupport.nextSecondarySeed() >= 0)
                    --spins;
            }//队列为空，初始化操作
            else if ((p = wtail) == null) { // initialize queue
                //新建节点，前置节点为空
                WNode hd = new WNode(WMODE, null);
                //CAS更新 新节点为头结点，成功后更新新节点为尾节点
                if (U.compareAndSwapObject(this, WHEAD, null, hd))
                    wtail = hd;
            }//如果新增节点还未初始化，则新建，并设置前置节点为尾节点
            else if (node == null)
                node = new WNode(WMODE, p);
            else if (node.prev != p)
                //如果尾节点有变化，设置node的前置节点为尾节点
                node.prev = p;
            //CAS更新尾节点为新节点
            else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                //设置之前尾节点p的后继节点为新节点
                p.next = node;
                break;
            }
        }
        //自旋-阻塞并等待唤醒
        for (int spins = -1;;) {
            //h为头节点，np为新增节点的前置节点，pp为前前置节点，ps为前置节点的状态
            WNode h, np, pp; int ps;
            //如果头节点等于前置节点，说明快轮到自己了
            if ((h = whead) == p) {
                if (spins < 0)
                    //初始化自旋次数
                    spins = HEAD_SPINS;
                //增加自旋次数
                else if (spins < MAX_HEAD_SPINS)
                    spins <<= 1;
                //自旋获取锁
                for (int k = spins;;) { // spin at head
                    long s, ns;
                    //尝试获取写锁成功
                    if (((s = state) & ABITS) == 0L) {
                        if (U.compareAndSwapLong(this, STATE, s,
                                                 ns = s + WBIT)) {
                            //设置node为头节点
                            whead = node;
                            //前置节点置空
                            node.prev = null;
                            return ns;
                        }
                    }//获取锁失败，随机减自旋次数，当自旋次数减为0时跳出循环再重试
                    else if (LockSupport.nextSecondarySeed() >= 0 &&
                             --k <= 0)
                        break;
                }
            }//协助唤醒读节点
            else if (h != null) { // help release stale waiters
                WNode c; Thread w;
                //如果头节点的cowait链表（栈）不为空，唤醒里面的所有节点
                while ((c = h.cowait) != null) {
                    if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null)
                        U.unpark(w);
                }
            }
            //如果头节点没有变化
            if (whead == h) {
                //如果尾节点变化，更新尾节点
                if ((np = node.prev) != p) {
                    if (np != null)
                        (p = np).next = node;   // stale
                }//如果尾节点状态为0，更新状态为WAITING
                else if ((ps = p.status) == 0)
                    U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                //状态为取消，断开链表
                else if (ps == CANCELLED) {
                    if ((pp = p.prev) != null) {
                        node.prev = pp;
                        pp.next = node;
                    }
                }
                else {
                    //超时设置
                    long time; // 0 argument to park means no timeout
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        //已超时，剔除当前节点
                        return cancelWaiter(node, node, false);
                    Thread wt = Thread.currentThread();
                    //设置wt线程阻塞的对象是当前对象
                    U.putObject(wt, PARKBLOCKER, this);
                    //node的线程指向当前线程
                    node.thread = wt;
                    if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                        whead == h && node.prev == p)
                        //阻塞当前线程
                        U.park(false, time);  // emulate LockSupport.park
                    node.thread = null;
                    //线程唤醒后，设置wt阻塞的对象为null
                    U.putObject(wt, PARKBLOCKER, null);
                    //线程中断处理
                    if (interruptible && Thread.interrupted())
                        //取消等待
                        return cancelWaiter(node, node, true);
                }
            }
        }
    }
```
1. 如果头节点等于尾节点，说明没有其它线程排队，那就多自旋一会，看能不能尝试获取到写锁；
2. 否则，自旋次数为0，直接让其入队；

#### unlockWrite(long stamp)

```java
    public void unlockWrite(long stamp) {
        WNode h;
        //如果版本号有变化或锁状态为没有加锁
        if (state != stamp || (stamp & WBIT) == 0L)
            throw new IllegalMonitorStateException();
        //更新版本号加1，释放写锁
        state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
        //头结点不为空且状态不为0，调用release方法唤醒下一节点
        if ((h = whead) != null && h.status != 0)
            release(h);
    }
    private void release(WNode h) {
        if (h != null) {
            WNode q; Thread w;
            //修改节点状态为0
            U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
            //如果头节点的下一个节点为空或者其状态为已取消
            if ((q = h.next) == null || q.status == CANCELLED) {
                //从尾节点向前遍历找到一个可用的节点
                for (WNode t = wtail; t != null && t != h; t = t.prev)
                    if (t.status <= 0)
                        q = t;
            }
            //唤醒头节点下一节点的线程
            if (q != null && (w = q.thread) != null)
                U.unpark(w);
        }
    }    
```
1. 更新state值，释放写锁；
2. 版本号加1；
3. 唤醒下一等待节点线程；

### readLock()
获取读锁。
```java
    public long readLock() {
        long s = state, next;  // bypass acquireRead on common uncontended case
        // 没有写锁占用，并且读锁被获取的次数未达到最大值
        // 尝试原子更新读锁被获取的次数加1
        // 如果成功直接返回，如果失败调用acquireRead()方法
        return ((whead == wtail && (s & ABITS) < RFULL &&
                 U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
                next : acquireRead(false, 0L));
    }
```
#### acquireRead(boolean interruptible, long deadline)

```java
    private long acquireRead(boolean interruptible, long deadline) {
        WNode node = null, p;
        //自旋，入队
        for (int spins = -1;;) {
            WNode h;
            //如果头节点等于尾节点,说明没有排队的线程了，快轮到自己了，直接自旋不断尝试获取读锁
            if ((h = whead) == (p = wtail)) {
                for (long m, s, ns;;) {
                    //读锁个数没满，通过CAS更新s的值，并返回
                    if ((m = (s = state) & ABITS) < RFULL ?
                        U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                        (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                        return ns;
                    //m >= WBIT表示有其它线程先一步获取了写锁
                    else if (m >= WBIT) {
                        if (spins > 0) {
                            //随机立减自旋次数
                            if (LockSupport.nextSecondarySeed() >= 0)
                                --spins;
                        }
                        else {
                            //自旋次数为0
                            if (spins == 0) {
                                WNode nh = whead, np = wtail;
                                //跳出
                                if ((nh == h && np == p) || (h = nh) != (p = np))
                                    break;
                            }
                            //设置自旋次数
                            spins = SPINS;
                        }
                    }
                }
            }
            //尾节点为空，初始化链表
            if (p == null) { // initialize queue
                //初始化新节点，CAS更新头结点
                WNode hd = new WNode(WMODE, null);
                if (U.compareAndSwapObject(this, WHEAD, null, hd))
                    //同时设置新节点为尾节点
                    wtail = hd;
            }//node节点为空，初始化
            else if (node == null)
                //p作为新节点的前置节点
                node = new WNode(RMODE, p);
            //如果头节点等于尾节点或尾节点不是读模式
            else if (h == p || p.mode != RMODE) {
                //入队操作，更新node为尾节点
                if (node.prev != p)
                    node.prev = p;
                else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                    p.next = node;
                    break;
                }
            }//是尾节点为读模式，将当前节点加入到尾节点的cowait中，这是一个栈
            else if (!U.compareAndSwapObject(p, WCOWAIT,
                                             node.cowait = p.cowait, node))
                //CAS失败到达这里
                node.cowait = null;
            else {
                //阻塞当前线程并等待被唤醒
                for (;;) {
                    WNode pp, c; Thread w;
                    //如果头节点不为空且其cowait不为空，协助唤醒其中等待的读线程
                    if ((h = whead) != null && (c = h.cowait) != null &&
                        U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null) // help release
                        U.unpark(w);
                    //如果头节点等于前置节点的前置或头结点等于前置节点或者前置节点的前置为空
                    if (h == (pp = p.prev) || h == p || pp == null) {
                        long m, s, ns;
                        do {
                            //尝试获取锁
                            if ((m = (s = state) & ABITS) < RFULL ?
                                U.compareAndSwapLong(this, STATE, s,
                                                     ns = s + RUNIT) :
                                (m < WBIT &&
                                 (ns = tryIncReaderOverflow(s)) != 0L))
                                return ns;
                        //当前时刻没有其它线程占有写锁就不断尝试
                        } while (m < WBIT);
                    }
                    //如果头节点没有变化且前置节点的前置没有变化
                    if (whead == h && p.prev == pp) {
                        long time;
                        //如果前置节点的前置为空或头节点等于前置节点或前置节点已取消
                        if (pp == null || h == p || p.status > 0) {
                            //跳出
                            node = null; // throw away
                            break;
                        }
                        //处理超时
                        if (deadline == 0L)
                            time = 0L;
                        else if ((time = deadline - System.nanoTime()) <= 0L)
                            //如果超时了，取消当前节点
                            return cancelWaiter(node, p, false);
                        Thread wt = Thread.currentThread();
                        //设置阻塞当前线程的对象为this
                        U.putObject(wt, PARKBLOCKER, this);
                        node.thread = wt;
                        //检测之前的条件是否变化
                        if ((h != pp || (state & ABITS) == WBIT) &&
                            whead == h && p.prev == pp)
                            //阻塞当前线程
                            U.park(false, time);
                        
                        //当前线程被唤醒，清除线程
                        node.thread = null;
                        U.putObject(wt, PARKBLOCKER, null);
                        //检测中断，取消节点
                        if (interruptible && Thread.interrupted())
                            return cancelWaiter(node, p, true);
                    }
                }
            }
        }
        //只有第一个读线程会走到下面的for循环处
        for (int spins = -1;;) {
            WNode h, np, pp; int ps;
            //头结点等于尾节点，不断尝试获取锁
            if ((h = whead) == p) {
                //设置自旋次数
                if (spins < 0)
                    spins = HEAD_SPINS;
                //自旋次数扩容
                else if (spins < MAX_HEAD_SPINS)
                    spins <<= 1;
                //自旋获取锁
                for (int k = spins;;) { // spin at head
                    long m, s, ns;
                    if ((m = (s = state) & ABITS) < RFULL ?
                        U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                        (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                        //获取成功
                        WNode c; Thread w;
                        whead = node;
                        node.prev = null;
                        //唤醒当前节点中所有等待着的读线程
                        // 当前节点是第一个读节点，其它读节点都挂这个节点的cowait栈中
                        while ((c = node.cowait) != null) {
                            if (U.compareAndSwapObject(node, WCOWAIT,
                                                       c, c.cowait) &&
                                (w = c.thread) != null)
                                U.unpark(w);
                        }
                        return ns;
                    }//如果当前有其它线程占有着写锁，并且没有自旋次数了，跳出当前循环
                    else if (m >= WBIT &&
                             LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                        break;
                }
            }//如果头节点不等待尾节点且不为空且其为读模式，协助唤醒里面的读线程
            else if (h != null) {
                WNode c; Thread w;
                while ((c = h.cowait) != null) {
                    if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                        (w = c.thread) != null)
                        U.unpark(w);
                }
            }
            //如果头节点没有变化
            if (whead == h) {
                //更新前置节点及其状态等
                if ((np = node.prev) != p) {
                    if (np != null)
                        (p = np).next = node;   // stale
                }
                else if ((ps = p.status) == 0)
                    U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
                else if (ps == CANCELLED) {
                    if ((pp = p.prev) != null) {
                        node.prev = pp;
                        pp.next = node;
                    }
                }
                else {
                    //第一个读节点即将进入阻塞
                    long time;
                    //超时处理
                    if (deadline == 0L)
                        time = 0L;
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        //超时取消当前节点
                        return cancelWaiter(node, node, false);
                    Thread wt = Thread.currentThread();
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    if (p.status < 0 &&
                        (p != h || (state & ABITS) == WBIT) &&
                        whead == h && node.prev == p)
                        //阻塞第一个读节点并等待被唤醒
                        U.park(false, time);
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, node, true);
                }
            }
        }
    }
```
1. 读节点进来都是先判断是头节点如果等于尾节点，说明快轮到自己了，就不断地尝试获取读锁，如果成功了就返回；
2. 如果头节点不等于尾节点，这里就会让当前节点入队，这里入队又分成了两种；
3. 一种是首个读节点入队，它是会排队到整个队列的尾部，然后跳出第一段自旋；
4. 另一种是非第一个读节点入队，它是进入到首个读节点的cowait栈中，所以更确切地说应该是入栈；
5. 不管是入队还入栈后，都会再次检测头节点是不是等于尾节点了，如果相等，则会再次不断尝试获取读锁；
6. 如果头节点不等于尾节点，那么才会真正地阻塞当前线程并等待被唤醒；
7. 上面说的首个读节点其实是连续的读线程中的首个，如果是两个读线程中间夹了一个写线程，还是老老实实的排队。

#### unlockRead(long stamp)

```java
    public void unlockRead(long stamp) {
        long s, m; WNode h;
        for (;;) {
            //状态校验
            if (((s = state) & SBITS) != (stamp & SBITS) ||
                (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)
                throw new IllegalMonitorStateException();
            //读线程个数正常
            if (m < RFULL) {
                //释放一次读锁
                if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                    //如果读锁全部释放了且头节点不为空且状态不为0，唤醒它的下一个节点
                    if (m == RUNIT && (h = whead) != null && h.status != 0)
                        release(h);
                    break;
                }
            }//读线程个数溢出检测
            else if (tryDecReaderOverflow(s) != 0L)
                break;
        }
    }
    private void release(WNode h) {
        if (h != null) {
            WNode q; Thread w;
            //CAS更新节点状态
            U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
            //如果头结点下一节点为空或状态为已取消
            if ((q = h.next) == null || q.status == CANCELLED) {
                //从尾节点遍历可用节点
                for (WNode t = wtail; t != null && t != h; t = t.prev)
                    if (t.status <= 0)
                        q = t;
            }
            //唤醒下一节点线程
            if (q != null && (w = q.thread) != null)
                U.unpark(w);
        }
    }    
```
### tryOptimisticRead()
乐观读。
```java
    //如果没有写锁就返回s & SBITS，否则返回0
    public long tryOptimisticRead() {
        long s;
        return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
    }
```
### validate(long stamp)
检查锁状态是否发生变化。
```java
    public boolean validate(long stamp) {
        U.loadFence();
        return (stamp & SBITS) == (state & SBITS);
    }
```
## 总结
1. `StampedLock`也是一种读写锁，它不是基于AQS实现的；
2. `StampedLock`相较于`ReentrantReadWriteLock`多了一种乐观读的模式，以及读锁转化为写锁的方法；
3. `StampedLock`的state存储的是版本号，确切地说是高24位存储的是版本号，写锁的释放会增加其版本号，读锁不会；
4. `StampedLock`的低7位存储的读锁被获取的次数，第8位存储的是写锁被获取的次数；
5. `StampedLock`不是可重入锁，因为只有第8位标识写锁被获取了，并不能重复获取；
6. `StampedLock`中获取锁的过程使用了大量的自旋操作，对于短任务的执行会比较高效，长任务的执行会浪费大量CPU；
7. `StampedLock`不能实现条件锁；
### 对比

`StampedLock`与`ReentrantReadWriteLock`作为两种不同的读写锁方式，如下异同点：

1. 两者都有获取读锁、获取写锁、释放读锁、释放写锁的方法，这是相同点；
2. 两者的结构基本类似，都是使用state + CLH队列；
3. 前者的state分成三段，高24位存储版本号、低7位存储读锁被获取的次数、第8位存储写锁被获取的次数；
4. 后者的state分成两段，高16位存储读锁被获取的次数，低16位存储写锁被获取的次数；
5. 前者的CLH队列可以看成是变异的CLH队列，连续的读线程只有首个节点存储在队列中，其它的节点存储的首个节点的cowait栈中；
6. 后者的CLH队列是正常的CLH队列，所有的节点都在这个队列中；
7. 前者获取锁的过程中有判断首尾节点是否相同，也就是是不是快轮到自己了，如果是则不断自旋，所以适合执行短任务；
8. 后者获取锁的过程中非公平模式下会做有限次尝试；
9. 前者只有非公平模式，一上来就尝试获取锁；
10. 前者唤醒读锁是一次性唤醒连续的读锁的，而且其它线程还会协助唤醒；
11. 后者是一个接着一个地唤醒的；
12. 前者有乐观读的模式，乐观读的实现是通过判断state的高25位是否有变化来实现的；
13. 前者各种模式可以互转，类似tryConvertToXxx()方法；
14. 前者写锁不可重入，后者写锁可重入；
15. 前者无法实现条件锁，后者可以实现条件锁；
