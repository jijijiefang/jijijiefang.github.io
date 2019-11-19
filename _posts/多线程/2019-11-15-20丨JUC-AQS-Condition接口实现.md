---
layout:     post
title:      "20丨JUC-AQS-Condition接口实现"
date:       2019-11-15 13:41:42
author:     "jiefang"
header-style: text
tags:
    - 多线程
    - JUC
---
# JUC-AQS-Condition接口实现
## Condition简介
Condition是关联在Lock上的条件，提供对线程更加灵活和详细的唤醒和等待操作。
Condition与Object.wait和Object.notify的比较：

对比项 | Object监视器方法|Condition
---|---|---
前置条件 | 获取对象的锁|调用Lock.lock()获取锁</br>调用Lock.newCondition()获取Condition对象
调用方式| 直接调用|直接调用
等待队列个数|一个|多个
当前线程释放锁并进入等待状态|支持|支持
当前线程释放锁并进入等待状态，在等待状态不响应中断|不支持|支持
当前线程释放锁并进入超时等待状态|支持|支持
当前线程释放锁并进入从等待状态到将来的某个时间|不支持|支持
唤醒等待队列中的某个线程|支持|支持
唤醒等待队列中的全部线程|支持|支持

Condition是一种广义上的条件队列。他为线程提供了一种更为灵活的等待/通知模式，线程在调用await方法后执行挂起操作，直到线程等待的某个条件为真时才会被唤醒。Condition必须要配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此Condition一般都是作为Lock的内部实现。

wait/notify机制来类比await/signal机制：

- 调用wait方法的线程首先必须是已经进入了同步代码块，即已经获取了`监视器锁`；与之类似，调用await方法的线程首先必须获得`lock锁`;
- 调用wait方法的线程会释放已经获得的监视器锁，进入当前监视器锁的等待队列（`wait set`）中；与之类似，调用await方法的线程会释放已经获得的lock锁，进入到当前Condtion对应的`条件队列`中。
- 调用监视器锁的notify方法会唤醒等待在该监视器锁上的线程，这些线程将开始参与锁竞争，并在获得锁后，从`wait`方法处恢复执行；与之类似，调用Condtion的signal方法会唤醒对应的条件队列中的线程，这些线程将开始参与锁竞争，并在获得锁后，从`await`方法处开始恢复执行。

## 示例
使用Condition实现生产者和消费者：
```
class BoundedBuffer{
    final ReentrantLock lock = new ReentrantLock();
    final Condition producer = lock.newCondition();
    final Condition consumer = lock.newCondition();
    final Object[] items = new Object[10];
    int putptr, takeptr, count;
    public void put(Object x) throws InterruptedException{
        System.out.println("开始生产");
        lock.lock();
        try {
            while (count == items.length){
                System.out.println("满了");
                producer.await();
            }
            items[putptr] = x;
            if (++putptr == items.length) {
                putptr = 0;
            }
            ++count;
            consumer.signal();
        }catch (Exception e){

        }finally {
            lock.unlock();
        }
    }

    public Object take()throws InterruptedException{
        System.out.println("开始消费");
        Object x = null;
        lock.lock();
        try {
            while (count == 0){
                System.out.println("空了");
                consumer.await();
            }
            x = items[takeptr];
            if(++takeptr == items.length){
                takeptr = 0;
            }
            --count;
            producer.signal();
        }catch (Exception e){

        }finally {
            lock.unlock();
        }
        return x;
    }
}
```
## 原理
### 同步队列和条件队列
#### 同步队列
![同步队列](https://s2.ax1x.com/2019/11/04/KvhVRx.png)
#### 条件队列
每创建一个Condtion对象就会对应一个Condtion队列，每一个调用了Condtion对象的await方法的线程都会被包装成Node扔进一个条件队列中。

![条件队列](https://s2.ax1x.com/2019/11/14/MNWW60.png)
`condition queue`是一个单向链表，在该链表中使用`nextWaiter`属性来串联链表。但是，就像在`sync queue`中不会使用`nextWaiter`属性来串联链表一样，在condition queue中，也并不会用到prev, next属性，它们的值都为null。也就是说，在条件队列中，Node节点真正用到的属性只有三个：
- `thread`：代表当前正在等待某个条件的线程
- `waitStatus`：条件的等待状态
- `nextWaiter`：指向条件队列中的下一个节点

waitStatus取值：
```
volatile int waitStatus;
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;
```
`CONDITION`表示线程处于正常的等待状态，而只要waitStatus不是CONDITION，就认为线程不再等待了，此时就要从条件队列中出队。

- `condition queue`:入队时已经持有了锁 -> 在队列中释放锁 -> 离开队列时没有锁 -> 转移到`sync queue`
- `sync queue`：入队时没有锁 -> 在队列中争锁 -> 离开队列时获得了锁

## CondtionObject
AQS对Condition这个接口的实现主要是通过ConditionObject，，它的核心实现就是是一个条件队列，每一个在某个condition上等待的线程都会被封装成Node对象扔进这个条件队列。
### 核心属性
```
public class ConditionObject implements Condition, java.io.Serializable {
    private transient Node firstWaiter;

    private transient Node lastWaiter;
}
```
### 构造函数
条件队列是延时初始化的
```
public ConditionObject() { }
```
## Condition接口方法实现

### await()
```
//释放锁，并进入等待队列
public final void await() throws InterruptedException {
    // 如果当前线程在调动await()方法前已经被中断了，则直接抛出
    if (Thread.interrupted())
        throw new InterruptedException();
    //添加一个新的Condition Node 进入条件等待队列
    Node node = addConditionWaiter();
    //释放锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 如果当前队列不在同步队列中，说明刚刚被await, 还没有人调用signal方法，则直接将当前线程挂起
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //如果取到锁且线程不是因中断而唤醒
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //线程争抢到了锁，清除ondition queue中状态不是CONDITION的节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
具体实现步骤：

- 将当前线程封装成Node扔进条件队列中;
- 在节点被成功添加到队列的末尾后，调用fullyRelease来释放当前线程所占用的锁;
- 在当前线程的锁被完全释放了之后，调用LockSupport.park(this)把当前线程挂起;
- 判断等待期间中断状态
    - 0：代表整个过程中一直没有中断发生
    - `THROW_IE`: 表示退出`await()`方法时需要抛出`InterruptedException`，这种模式对应于**中断发生在signal之前**
    - REINTERRUPT: 表示退出`await()`方法时只需要再自我中断以下，这种模式对应于中断发生在signal之后，即**中断来的太晚了**。


#### addConditionWaiter
```
//添加一个新的Condition进入条件等待队列
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        //从头节点开始遍历整个队列，剔除其中waitStatus不为Node.CONDTION的节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```
`sync queue`与`condition queue`入队的区别：
- 节点加入`sync queue`时`waitStatus`的值为0;节点加入`condition queue`时`waitStatus`的值为`Node.CONDTION`。
- `sync queue`的头节点为dummy节点，如果队列为空，则会先创建一个dummy节点，再创建一个代表当前节点的Node添加在dummy节点的后面；而`condtion queue` 没有dummy节点，初始化时，直接将`firstWaiter`和`lastWaiter`直接指向新建的节点就行了。
- `sync queue`是一个双向队列，在节点入队后，要同时修改当前节点的前驱和前驱节点的后继；而在`condtion queue`中，我们只修改了前驱节点的`nextWaiter`,也就是说，`condtion queue`是作为单向队列来使用的。

#### fullyRelease
```
//完全释放锁
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        //全部释放
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```
#### isOnSyncQueue
挂起当前线程之前先用`isOnSyncQueue`确保了它不在`sync queue`中:
```
//判断是否在同步队列中
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    return findNodeFromTail(node);
}
//如果节点在同步队列中（从尾向后搜索），则返回true
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```
#### checkInterruptWhileWaiting
判断中断状态：
```
private static final int REINTERRUPT =  1;
private static final int THROW_IE    = -1;

private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
//中断状态为true时，判断线程是否被sign唤醒
final boolean transferAfterCancelledWait(Node node) {
    //一个节点的waitStatus还是Node.CONDITION，那就说明它还没有被signal过,则添加至sync queue
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    //进入sync queue 说明线程已经被sign唤醒
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```
汇报中断状态：
```
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```
#### await()总结

- 进入await()时必须是已经持有了锁
- 离开await()时同样必须是已经持有了锁
- 调用await()会使得当前线程被封装成Node扔进条件队列，然后释放所持有的锁
- 释放锁后，当前线程将在condition queue中被挂起，等待signal或者中断
- 线程被唤醒后会将会离开condition queue进入sync queue中进行抢锁
- 若在线程抢到锁之前发生过中断，则根据中断发生在signal之前还是之后记录中断模式
- 线程在抢到锁后进行善后工作（离开condition queue, 处理中断异常）
- 线程已经持有了锁，从await()方法返回

![image](https://s2.ax1x.com/2019/11/14/MU85EF.png)

**中断和signal所起到的作用都是将线程从`condition queue`中移除，加入到`sync queue`中去争锁，所不同的是，signal方法被认为是正常唤醒线程，中断方法被认为是非正常唤醒线程，如果中断发生在signal之前，则我们在最终返回时，应当抛出`InterruptedException`；如果中断发生在signal之后，我们就认为线程本身已经被正常唤醒了，这个中断来的太晚了，我们直接忽略它，并在`await()`返回时再自我中断一下，这种做法相当于将中断推迟至`await()`返回时再发生。**
### signalAll

- 调用signalAll方法的线程本身是已经持有了锁，现在准备释放锁了；
- 在条件队列里的线程是已经在对应的条件上挂起了，等待着被signal唤醒，然后去争锁。

signalAll源码：
- 将条件队列清空（只是令lastWaiter = firstWaiter = null，队列中的节点和连接关系仍然还存在）
- 将条件队列中的头节点取出，使之成为孤立节点(nextWaiter,prev,next属性都为null)
- 如果该节点处于被Cancelled了的状态，则直接跳过该节点（由于是孤立节点，则会被GC回收）
- 如果该节点处于正常状态，则通过enq方法将它添加到sync queue的末尾
- 判断是否需要将该节点唤醒(包括设置该节点的前驱节点的状态为SIGNAL)，如有必要，直接唤醒该节点
- 重复2-5，直到整个条件队列中的节点都被处理完
```
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
```
首先，与调用notify时线程必须是已经持有了监视器锁类似，在调用condition的signal方法时，线程也必须是已经持有了lock锁：

通过`isHeldExclusively`方法来实现,该方法由继承AQS的子类来实现，例如，`ReentrantLock`对该方法的实现为：
```
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```
接下来先通过firstWaiter是否为空判断条件队列是否为空，如果条件队列不为空，则调用doSignalAll方法：
```
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```
通过`lastWaiter = firstWaiter = null`;将整个条件队列清空，然后通过一个do-while循环，将原先的条件队列里面的节点一个一个拿出来(令nextWaiter = null)，再通过`transferForSignal`方法一个一个添加到`sync queue`的末尾：
```
final boolean transferForSignal(Node node) {
    //如果该节点在调用signal方法前已经被取消了，则直接跳过这个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 如果该节点在条件队列中正常等待，则利用enq方法将该节点添加至sync queue队列的尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    //只要前驱节点处于被取消的状态或者无法将前驱节点的状态修成Node.SIGNAL，那我们就将Node所代表的线程唤醒
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
#### signal()
signal()方法只会唤醒一个节点，对于AQS的实现来说，就是唤醒条件队列中第一个没有被Cancel的节点。

signal源码
```
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```
这个方法也是一个do-while循环，目的是遍历整个条件队列，找到第一个没有被cancelled的节点，并将它添加到条件队列的末尾。如果条件队列里面已经没有节点了，则将条件队列清空（firstWaiter=lasterWaiter=null）。

在这里，用的依然用的是`transferForSignal`方法，但是用到了它的返回值，只要节点被成功添加到sync queue中，`transferForSignal`就返回true, 此时while循环的条件就不满足了，整个方法就结束了，即调用signal()方法，只会唤醒一个线程。

总结： 调用signal()方法会从当前条件队列中取出第一个没有被cancel的节点添加到sync队列的末尾。
