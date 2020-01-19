---
layout:     post
title:      "Java集合-19丨SynchronousQueue"
date:       2019-12-18 22:24:10
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - 多线程
---
# SynchronousQueue
## 简介
Java注释
>A {@linkplain BlockingQueue blocking queue} in which each insert operation must wait for a corresponding remove operation by another thread, and vice versa.  A synchronous queue does not have any internal capacity, not even a capacity of one.  You cannot {@code peek} at a synchronous queue because an element is only present when you try to remove it; you cannot insert an element (using any method) unless another thread is trying to remove it;you cannot iterate as there is nothing to iterate.  The <em>head</em> of the queue is the element that the first queued inserting thread is trying to add to the queue; if there is no such queued thread then no element is available for removal and {@code poll()} will return {@code null}.  For purposes of other {@code Collection} methods (for example {@code contains}), a {@code SynchronousQueue} acts as an empty collection.  This queue does not permit {@code null} elements.

翻译
>{@linkplain BlockingQueue阻塞队列}，其中每个插入操作必须等待另一个线程进行相应的删除操作，反之亦然；同步队列没有任何内部容量，甚至没有一个容量。您不能{@ code peek}在同步队列中，因为当您尝试删除一个元素时，该元素仅存在；您不能插入元素（使用任何方法），除非另一个线程试图将其删除； 您无法迭代，因为没有要迭代的内容；队列的<em>head</em>是第一个排队的插入线程试图添加到队列中的元素；如果没有这样的排队线程，则没有可用于删除的元素，并且{@code poll（）}将返回{@code null}。出于其他{@code Collection}方法（例如{@code contains}）的目的， {@code SynchronousQueue}用作空集合。此队列不允许{@code null}元素。

**`SynchronousQueue`是java并发包下无缓冲阻塞队列，它用来在两个线程之间移交元素。**

### 类图

![SynchronousQueue](https://s2.ax1x.com/2019/12/18/Q7Nnr4.png)

## 源码

### 属性

```
    //CPU核数
    static final int NCPUS = Runtime.getRuntime().availableProcessors();
    //有超时的情况自旋多少次，当CPU数量小于2的时候不自旋
    static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;
    //没有超时的情况自旋多少次
    static final int maxUntimedSpins = maxTimedSpins * 16;
    // 针对有超时的情况，自旋了多少次后，如果剩余时间大于1000纳秒就使用带时间的LockSupport.parkNanos()这个方法
    static final long spinForTimeoutThreshold = 1000L;
    //传输器，即两个线程交换元素使用的东西
    private transient volatile Transferer<E> transferer;    
```
### 内部类

#### Transferer<E>
Transferer抽象类，主要定义了一个transfer方法用来传输元素。
```
    abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }
```
#### TransferQueue<E>
以队列方式实现的Transferer。
```
static final class TransferQueue<E> extends Transferer<E> {
    //队列节点
    static final class QNode {
        //下一个节点
        volatile QNode next;          // next node in queue
        //存储元素
        volatile Object item;         // CAS'ed to or from null
        //等待线程
        volatile Thread waiter;       // to control park/unpark
        //是否是数据节点
        final boolean isData;
    }
    //队列头节点
    transient volatile QNode head;
    //队列尾节点
    transient volatile QNode tail;
    //已取消节点
    transient volatile QNode cleanMe;  
}
```
#### TransferStack<E>
以栈方式实现的Transferer。
```
static final class TransferStack<E> extends Transferer<E> {
    //消费者（请求数据的）
    static final int REQUEST    = 0;
    //生产者（提供数据的）
    static final int DATA       = 1;
    //二者正在匹配中
    static final int FULFILLING = 2;
    //栈中节点
    static final class SNode {
        //下一节点
        volatile SNode next;        // next node in stack
        //匹配者
        volatile SNode match;       // the node matched to this
        //等待着的线程
        volatile Thread waiter;     // to control park/unpark
        //元素
        Object item;                // data; or null for REQUESTs
        //节点的类型，是消费者，是生产者，还是正在匹配中
        int mode;
    }
}
```
1. 定义了一个抽象类Transferer，里面定义了一个传输元素的方法；
2. 有两种传输元素的方法，一种是栈，一种是队列；
3. 栈的特点是后进先出，队列的特点是先进行出；
4. 栈只需要保存一个头节点就可以了，因为存取元素都是操作头节点；
5. 队列需要保存一个头节点一个尾节点，因为存元素操作尾节点，取元素操作头节点；
6. 每个节点中保存着存储的元素、等待着的线程，以及下一个节点；

### 构造方法
#### SynchronousQueue()
无参构造方法，默认非公平。
```
    public SynchronousQueue() {
        this(false);
    }
```
#### SynchronousQueue(boolean fair)
根据传入是否公平构造，公平模式是队列，非公平模式的是栈。
```
    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }
```
### 入队

#### add(E e)
添加元素成功返回true，失败抛异常。
```
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```
#### offer(E e)
添加元素成功返回true，否则返回false。
```
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        return transferer.transfer(e, true, 0) != null;
    }
```
#### put(E e)
添加元素失败，线程中断。
```
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        //三个参数分别是：传输的元素，是否需要超时，超时的时间
        if (transferer.transfer(e, false, 0) == null) {
            //如果传输失败，直接让线程中断并抛出中断异常
            Thread.interrupted();
            throw new InterruptedException();
        }
    }
```
#### offer(E e, long timeout, TimeUnit unit)

```
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (e == null) throw new NullPointerException();
        if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
            return true;
        if (!Thread.interrupted())
            return false;
        throw new InterruptedException();
    }
```
### 出队

#### remove()
```
    public E remove() {
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }
```
#### poll()
```
    public E poll() {
        return transferer.transfer(null, true, 0);
    }
```
#### poll(long timeout, TimeUnit unit)
```
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = transferer.transfer(null, true, unit.toNanos(timeout));
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }
```
#### take()
```
    public E take() throws InterruptedException {
        //三个参数分别是：null，是否需要超时，超时的时间
        //第一个参数为null表示是消费者，要取元素
        E e = transferer.transfer(null, false, 0);
        //如果取到元素就返回
        if (e != null)
            return e;
        //线程中断并抛异常
        Thread.interrupted();
        throw new InterruptedException();
    }
```

### Transferer<E>.transfer
transfer()方法同时实现了取元素和放元素的功能。
```
    //e为null表示是消费者，要取元素；否则是生产者，e是生产的元素
    abstract E transfer(E e, boolean timed, long nanos);
```
#### TransferStack<E>.transfer
基本算法是循环尝试以下三个动作之一：
1. 如果显然是空的或已经包含相同模式的节点，尝试将节点压入堆栈并等待匹配，返回它，如果取消则返回null；
2. 如果显然包含互补模式的节点，则尝试将满足要求的节点压入堆栈，将与相应的等待节点进行匹配，从堆栈中弹出两者，然后返回匹配项。由于其他线程正在执行操作3，因此实际上可能不需要匹配或取消链接：
3. 如果栈顶已经包含另一个充实的节点，请通过执行其match和/或pop操作来帮助它，然后继续。帮助的代码本质上与实现相同，只是不返回该项目。
```
    E transfer(E e, boolean timed, long nanos) {

        SNode s = null; // constructed/reused as needed
        //根据传入元素是否是null，判断生产者和消费者
        int mode = (e == null) ? REQUEST : DATA;

        for (;;) {
            //栈顶元素
            SNode h = head;
            //栈顶没有元素，或者栈顶元素跟当前元素是一个模式的
            //都是生产者节点或者都是消费者节点
            if (h == null || h.mode == mode) {  // empty or same-mode
                //超时而且已到期
                if (timed && nanos <= 0) {      // can't wait
                    //如果头节点不为空且是取消状态
                    if (h != null && h.isCancelled())
                        //弹出取消的节点
                        casHead(h, h.next);     // pop cancelled node
                    else//否则返回null
                        return null;
                //包装e为栈节点，入栈，CAS更新e的节点为栈顶
                } else if (casHead(h, s = snode(s, e, h, mode))) {
                    //调用awaitFulfill()方法自旋+阻塞当前入栈的线程并等待被匹配到
                    SNode m = awaitFulfill(s, timed, nanos);
                  //如果m等于s，说明取消了，那么就把它清除掉，并返回null
                    if (m == s) {               // wait was cancelled
                        clean(s);
                        return null;
                    }
                    // 如果头节点不为空，并且头节点的下一个节点是s
                    // 就把头节点换成s的下一个节点
                    // 就是h和s都弹出了,也就是栈顶两个元素都弹出了
                    if ((h = head) != null && h.next == s)
                        casHead(h, s.next);     // help s's fulfiller
                    //根据当前节点的模式判断返回m还是s中的值
                    return (E) ((mode == REQUEST) ? m.item : s.item);
                }
            // 到这里说明头节点和当前节点模式不一样
            // 如果头节点不是正在匹配中
            } else if (!isFulfilling(h.mode)) { // try to fulfill
                //栈顶元素已取消，更新栈顶为下一个节点
                if (h.isCancelled())            // already cancelled
                    casHead(h, h.next);         // pop and retry
                //头节点没有在匹配中，就让当前节点先入队，再让他们尝试匹配
                // 且s成为了新的头节点，它的状态是正在匹配中
                else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                    for (;;) { // loop until matched or waiters disappear
                        SNode m = s.next;       // m is s's match
                        // 如果m为null，说明除了s节点外的节点都被其它线程先一步匹配掉了
                        //就清空栈并跳出内部循环，到外部循环再重新入栈判断                        
                        if (m == null) {        // all waiters are gone
                            casHead(s, null);   // pop fulfill node
                            s = null;           // use new node next time
                            break;              // restart main loop
                        }
                        SNode mn = m.next;
                        
                        if (m.tryMatch(s)) {
                            casHead(s, mn);     // pop both s and m
                            return (E) ((mode == REQUEST) ? m.item : s.item);
                        } else                  // lost match
                            //尝试匹配失败，说明m已经先一步被其它线程匹配了,就协助清除它
                            s.casNext(m, mn);   // help unlink
                    }
                }
            //到这里说明当前节点和头节点模式不一样
            //且头节点是正在匹配中
            } else {                            // help a fulfiller
                SNode m = h.next;               // m is h's match
                //如果m为null，说明m已经被其它线程先一步匹配了
                if (m == null)                  // waiter is gone
                    casHead(h, null);           // pop fulfilling node
                else {
                    SNode mn = m.next;
                   //协助匹配，如果m和s尝试匹配成功，就弹出栈顶的两个元素m和s
                    if (m.tryMatch(h))          // help match
                        // 将栈顶的两个元素弹出后，再让s重新入栈
                        casHead(h, mn);         // pop both h and m
                    else                        // lost match
                        //尝试匹配失败，说明m已经先一步被其它线程匹配了,协助清除它
                        h.casNext(m, mn);       // help unlink
                }
            }
        }
    }
    SNode awaitFulfill(SNode s, boolean timed, long nanos) {
        //到期时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        //当前线程
        Thread w = Thread.currentThread();
        //自旋次数
        int spins = (shouldSpin(s) ?
                     (timed ? maxTimedSpins : maxUntimedSpins) : 0);
        for (;;) {
            //当前线程中断，取消等待
            if (w.isInterrupted())
                s.tryCancel();
            //检查s是否匹配到了元素m（有可能是其它线程的m匹配到当前线程的s）
            SNode m = s.match;
            //如果匹配到了，直接返回m
            if (m != null)
                return m;
            //如果设置超时等待
            if (timed) {
                //检查超时时间如果小于0了，尝试清除s
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    s.tryCancel();
                    continue;
                }
            }
            //如果还有自旋次数，自旋次数减一，并进入下一次自旋
            if (spins > 0)
                spins = shouldSpin(s) ? (spins-1) : 0;
            else if (s.waiter == null)
                //如果s的waiter为null，把当前线程注入进去，并进入下一次自旋
                s.waiter = w; // establish waiter so can park next iter
            //如果不允许超时，直接阻塞，并等待被其它线程唤醒，唤醒后继续自旋并查看是否匹配到了元素
            else if (!timed)
                LockSupport.park(this);
            //如果允许超时且还有剩余时间，就阻塞相应时间
            else if (nanos > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanos);
        }
    }
    boolean tryMatch(SNode s) {
        //如果m还没有匹配者，就把s作为它的匹配者
        if (match == null &&
            UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
            Thread w = waiter;
            if (w != null) {    // waiters need at most one unpark
                waiter = null;
                // 唤醒m中的线程，两者匹配完毕
                LockSupport.unpark(w);
            }
            return true;
        }
        //可能其它线程先一步匹配了m，返回其是否是s
        return match == s;
    }    
```

#### TransferQueue<E>.transfer
基本算法是循环尝试执行以下两个操作之一：
1. 如果队列显然是空的或持有相同模式的节点，请尝试将节点添加到等待者的队列中，等待被实现（或取消）并返回匹配项。
2. 如果队列显然包含等待项，并且此调用是互补模式，请尝试通过CAS 等待节点的item字段并将其出队，然后返回匹配项来实现。
```
    E transfer(E e, boolean timed, long nanos) {

        QNode s = null; // constructed/reused as needed
        boolean isData = (e != null);

        for (;;) {
            QNode t = tail;
            QNode h = head;
            //链表头或尾为null，自旋
            if (t == null || h == null)         // saw uninitialized value
                continue;                       // spin
            //链表为空或同样模式
            if (h == t || t.isData == isData) { // empty or same-mode
                QNode tn = t.next;
                //链表尾节点已经变化，自旋
                if (t != tail)                  // inconsistent read
                    continue;
                //尾节点滞后，更新尾节点
                if (tn != null) {               // lagging tail
                    advanceTail(t, tn);
                    continue;
                }
                if (timed && nanos <= 0)        // can't wait
                    return null;
                if (s == null)//构建新节点
                    s = new QNode(e, isData);
                //CAS更新失败，自旋
                if (!t.casNext(null, s))        // failed to link in
                    continue;
                //CAS更新尾节点
                advanceTail(t, s);              // swing tail and wait
                //等待匹配，并返回匹配节点的item，如果取消等待则返回该节点s
                Object x = awaitFulfill(s, e, timed, nanos);
                if (x == s) {                   // wait was cancelled
                    //取消等待，清除s节点
                    clean(t, s);
                    return null;
                }
                //链表还没有断开
                if (!s.isOffList()) {           // not already unlinked
                    advanceHead(t, s);          // unlink if head
                    if (x != null)              // and forget fields
                        //指向自身
                        s.item = s;
                    //等待线程为null
                    s.waiter = null;
                }
                return (x != null) ? (E)x : e;

            } else {                            // complementary-mode
                QNode m = h.next;               // node to fulfill
                if (t != tail || m == null || h != head)
                    continue;                   // inconsistent read

                Object x = m.item;
                if (isData == (x != null) ||    // m already fulfilled
                    x == m ||                   // m cancelled
                    !m.casItem(x, e)) {         // lost CAS
                    advanceHead(h, m);          // dequeue and retry
                    continue;
                }

                advanceHead(h, m);              // successfully fulfilled
                LockSupport.unpark(m.waiter);
                return (x != null) ? (E)x : e;
            }
        }
    }
```
## 总结
1. SynchronousQueue是java里的无缓冲队列，用于在两个线程之间直接移交元素；
2. SynchronousQueue有两种实现方式，一种是公平（队列）方式，一种是非公平（栈）方式；
3. 栈方式中的节点有三种模式：生产者、消费者、正在匹配中；
4. 栈方式的大致思路是如果栈顶元素跟自己一样的模式就入栈并等待被匹配，否则就匹配，匹配到了就返回；