---
layout:     post
title:      "AbstractQueuedSynchronizer源码分析-独占锁的获取"
date:       2019-11-04 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
    - JUC
---
# AbstractQueuedSynchronizer源码分析-独占锁的获取

## acquire(int arg)
>`独占模式`下获取资源/锁，忽略中断的影响。

```
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```

`acquire`方法流程如下：
- `tryAcquire`:由子类实现，尝试直接获取资源，如果成功则直接返回，失败进入第二步；
- `addWaiter`:获取资源失败后，将当前线程包装为独占模式的node加入等待队列的尾部，并返回这个node；
- `acquireQueued`:在前驱节点就是head节点的时候,继续尝试获取锁；将当前线程挂起,使CPU不再调度它
- `selfInterrupt`:如果线程在等待过程中被中断(interrupt)是不响应的，在获取资源成功之后根据返回的中断状态调用`selfInterrupt`方法再把中断状态补上。


### tryAcquire
```
//AQS中的tryAcquire，需要由子类覆盖
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```
>尝试获取资源，成功返回true。具体资源获取/释放方式交由子类实现。

`ReentrantLock`中公平锁和非公平锁的实现如下:
```
//非公平锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    //获取锁的状态
    int c = getState();
    //c=0说明锁没有被线程占用，可以获取；
    //非公平锁直接CAS方式获取锁，获取成功将自身线程设置为占用锁的线程。
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }//如果c>0，说明锁已经被占用
    //如果当前线程和占用锁的线程一样，则成为可重入锁，state+1
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //别的线程占用了锁，获取锁失败
    return false;
}
//公平锁
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //公平锁和非公平锁的区别在于：如果锁没有被占用，在获取锁时检查
        //是否存在排队的线程队列，若存在当前线程是否是head节点则为false
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
### addWaiter
获取独占锁失败后，将当前线程标记为独占模式并加入等待队列的尾部，返回插入的等待节点。
```
private Node addWaiter(Node mode) {
    //Node.EXCLUSIVE用于独占模式
    //nextWaiter属性被设为null
    /*Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }*/
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    //尝试快速入队：如果队列不为空，使用CAS直接将当前节点设置为尾节点
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //队列为空或CAS失败，初始化队列或循环入队
    enq(node);
    return node;
}
//快速入队失败或队列为空，通过自旋+CAS的方式，确保当前节点入队
private Node enq(final Node node) {
    for (;;) {
        //如果是空队列, 首先进行初始化
        //队列不是在构造的时候初始化的, 而是延迟到需要用的时候再初始化, 以提升性能
        Node t = tail;
        if (t == null) {
            //注意，初始化时使用new Node()方法新建了一个dummy节点
            if (compareAndSetHead(new Node()))
                //将尾节点指向dummy节点，并没有返回
                tail = head;
        } else {
            //队列已经不是空的了, 这个时候再继续尝试将节点加到队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
需要注意的是：
当队列为空时，初始化队列并没有使用当前传进来的节点，而是：
**新建了一个空节点**；</br>
在新建完空的头节点之后，**并没有立即返回**，而是将尾节点指向当前的头节点，然后进入下一轮循环。</br>
在下一轮循环中，尾节点已经不为null了，此时再将包装了当前线程的Node加到这个空节点后面。</br>
**head节点不代表任何线程，它就是一个空节点**。

#### 尾分叉
enq方法中一个比较有趣的现象，叫做尾分叉。</br>
将一个节点node添加到sync queue的末尾也需要三步：
- 设置node的前驱节点为当前的尾节点：`node.prev = t`
- 修改`tail`属性，使它指向当前节点
- 改原来的尾节点，使它的next指向当前节点

![image](https://s2.ax1x.com/2019/11/04/KvzYU1.png)

但是需要注意的，这三步并不是一个原子操作，第一步很容易成功；而第二步由于是一个CAS操作，在并发条件下有可能失败，第三步只有在第二步成功的条件下才执行。这里的CAS保证了同一时刻只有一个节点能成为尾节点，其他节点将失败，失败后将回到for循环中继续重试。</br>

当有大量的线程在同时入队的时候，同一时刻，只有一个线程能完整地完成这三步，而其他线程只能完成第一步，于是就出现了尾分叉：

![image](https://s2.ax1x.com/2019/11/04/KvzIbj.png)

注意，这里第三步是在第二步执行成功后才执行的，这就意味着，有可能即使已经完成了第二步，将新的节点设置成了尾节点，**此时原来旧的尾节点的next值可能还是null**(因为还没有来的及执行第三步)，如果此时有线程恰巧从头节点开始向后遍历整个链表，则它是遍历不到新加进来的尾节点的，但是这显然是不合理的，因为现在的tail已经指向了新的尾节点。</br>
另一方面，当我们完成了第二步之后，第一步一定是完成了的，所以如果我们从尾节点开始向前遍历，已经可以遍历到所有的节点。这也就是为什么在AQS相关的源码中，有时候常常会出现从尾节点开始逆向遍历链表——因为一个节点要能入队，则它的prev属性一定是有值的，但是它的next属性可能暂时还没有值。</br>

至于那些“分叉”的入队失败的其他节点，在下一轮的循环中，它们的prev属性会重新指向新的尾节点，继续尝试新的CAS操作，最终，所有节点都会通过自旋不断的尝试入队，直到成功为止。

### acquireQueued
说明:
- 能执行到该方法, 说明**addWaiter** 方法已经成功将包装了当前Thread的节点添加到了等待队列的队尾
- 该方法中将再次尝试去获取锁
- 在再次尝试获取锁失败后, 判断是否需要把当前线程挂起

再次尝试获取锁是基于一定的条件的,即:
>当前节点的前驱节点就是HEAD节点

head节点就是个哑节点，它不代表任何线程，或者代表了持有锁的线程，如果当前节点的前驱节点就是head节点，那就说明当前节点已经是排在整个等待队列最前面的了。
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //当前节点的前驱就是HEAD节点时, 再次尝试获取锁
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                //获取锁成功后，设置当前节点为头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //在获取锁失败后, 判断是否需要把当前线程挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
//将head指向传进来的node,并且将node的thread和prev属性置为null
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
//检查前驱节点的waitStatus值做不同的操作
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //如果前驱节点的waitStatus=-1,z则当前线程可以安全的park了。
    if (ws == Node.SIGNAL)
        return true;
    // 当前节点的 ws > 0, 则为 Node.CANCELLED 说明前驱节点已经取消了等待锁(由于超时或者中断等原因)
    //既然前驱节点不等了, 那就继续往前找, 直到找到一个还在等待锁的节点
    //跨过不等待锁的节点, 排在等待锁的节点的后面
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    //前驱节点的状态既不是SIGNAL，也不是CANCELLED,用CAS设置前驱节点的ws为 Node.SIGNAL,但是不立即park,在park前调用者需要重试来确认它不能获取资源
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
//阻塞当前线程，返回中断状态
private final boolean parkAndCheckInterrupt() {
    //线程被挂起，停在这里不再往下执行了
    LockSupport.park(this);
    return Thread.interrupted();
}
```
**setHead**()方法如图所示：

![image](https://s2.ax1x.com/2019/11/04/KxmG4S.md.png)

这个方法的本质是丢弃原来的head，将head指向已经获得了锁的node。但是接着又将该node的thread属性置为null了，**这某种意义上导致了这个新的head节点又成为了一个哑节点**，它不代表任何线程。为什么要这样做呢，因为在tryAcquire调用成功后，exclusiveOwnerThread属性就已经记录了当前获取锁的线程了，此处没有必要再记录。**这某种程度上就是将当前线程从等待队列里面拿出来了，是一个变相的出队操作**。

**shouldParkAfterFailedAcquire**所做的事情：

- 如果前驱节点的`waitStatus`值为 `Node.SIGNAL` 则直接返回 true。
- 如果前驱节点的`waitStatus`值为 `Node.CANCELLED` (ws > 0), 则跳过那些节点, 重新寻找正常等待中的前驱节点，然后排在它后面，返回false。
- 其他情况, 将前驱节点的状态改为 `Node.SIGNAL`, 返回false。

## 总结
- AQS中用**state**属性表示锁，如果能成功将state属性通过CAS操作从0设置成1即获取了锁。
- 获取了锁的线程才能将**exclusiveOwnerThread**设置成自己。
- **addWaiter**负责将当前等待锁的线程包装成Node,并成功地添加到队列的末尾，这一点是由它调用的enq方法保证的，enq方法同时还负责在队列为空时初始化队列。
- **acquireQueued**方法用于在Node成功入队后，继续尝试获取锁（取决于Node的前驱节点是不是head），或者将线程挂起。
- **shouldParkAfterFailedAcquire**方法用于保证当前线程的前驱节点的waitStatus属性值为SIGNAL,从而保证了自己挂起后，前驱节点会负责在合适的时候唤醒自己。
- **parkAndCheckInterrupt**方法用于挂起当前线程，并检查中断状态。
- 如果最终成功获取了锁，线程会从lock()方法返回，继续往下执行；否则，线程会阻塞等待。


[逐行分析AQS源码(1)——独占锁的获取](https://segmentfault.com/a/1190000015739343)

[JUC源码分析—AQS](https://www.jianshu.com/p/a8d27ba5db49)

[【死磕Java并发】—–J.U.C之AQS：CLH同步队列](http://cmsblogs.com/?p=2188)
