---
layout:     post
title:      "Java集合-20丨LinkedTransferQueue"
date:       2019-12-20 11:50:09
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - 多线程
---
# LinkedTransferQueue

## 简介
Java注释
>An unbounded {@link TransferQueue} based on linked nodes. This queue orders elements FIFO (first-in-first-out) with respect to any given producer. The <em>head</em> of the queue is that element that has been on the queue the longest time for some producer. The <em>tail</em> of the queue is that element that has been on the queue the shortest time for some producer.

翻译
>基于链接节点的无限制{@link TransferQueue}。此队列针对任何给定的生产者对元素FIFO（先进先出）进行排序。队列的<em> head</em>是该元素在队列中对于某些生产者而言最长的时间。队列的<em> tail </ em>是在某些生产者最短的时间内进入队列的元素。

`LinkedTransferQueue`采用一种预占模式。意思就是消费者线程取元素时，如果队列为空，那就生成一个节点（节点元素为null）入队，然后消费者线程被等待在这个节点上，后面生产者线程入队时发现有一个元素为null的节点，生产者线程就不入队了，直接就将元素填充到该节点，并唤醒该节点等待的线程，被唤醒的消费者线程取走元素，从调用的方法返回。

**LinkedTransferQueue是LinkedBlockingQueue、SynchronousQueue（公平模式）、ConcurrentLinkedQueue三者的集合体，它综合了这三者的方法，并且提供了更加高效的实现方式。**

### 类图

![LinkedTransferQueue](https://s2.ax1x.com/2019/12/19/QLCw80.png)

### 存储结构

![image](https://s2.ax1x.com/2019/12/19/QLA9lq.png)
- **unmatched node**：未被匹配的节点。可能是一个生产者节点（item不为null），也可能是一个消费者节点（item为null）。
- **matched node**：已经被匹配的节点。可能是一个生产者节点（item不为null）的数据已经被一个消费者拿走；也可能是一个消费者节点（item为null）已经被一个生产者填充上数据。

#### 双重队列
`LinkedTransferQueue`和典型的单向链表结构不同，它的Node 存储了一个isData的boolean型字段，也就是说它的节点可以代表一个数据或者是一个请求，称为`双重队列（Dual Queue）`。在消费者获取元素时，如果队列为空，当前消费者就会作为一个“元素为null”的节点被放入队列中等待，所以 `LinkedTransferQueue`中的节点存储了生产者节点（item不为null）和消费者节点（item为null），这两种节点就是通过isData来区分的。

![双重队列](https://s2.ax1x.com/2019/12/19/QLVKeK.png)

## 源码

### 属性

```
    //是否多核处理器
    private static final boolean MP =
        Runtime.getRuntime().availableProcessors() > 1;
    //自旋次数
    private static final int FRONT_SPINS   = 1 << 7;
    //前驱节点正在处理，当前节点需要自旋的次数
    private static final int CHAINED_SPINS = FRONT_SPINS >>> 1;
    //最多清除失败次数
    static final int SWEEP_THRESHOLD = 32;
    //头节点
    transient volatile Node head;
    //尾节点
    private transient volatile Node tail;
    //删除节点失败的次数
    private transient volatile int sweepVotes;
    //放取元素的几种方式：
    //不等待，直接返回匹配结果，用于poll()和tryTransfer()方法
    private static final int NOW   = 0; // for untimed poll, tryTransfer
    //异步操作，直接把元素添加到队列尾，不等待匹配。用在offer, put, add中。
    private static final int ASYNC = 1; // for offer, put, add
    //等待元素被消费者接收。用在transfer, take中。
    private static final int SYNC  = 2; // for transfer, take
    //超时，用于有超时的poll()和tryTransfer()方法中
    private static final int TIMED = 3; // for timed poll, tryTransfer 
```
### 内部类

```
static final class Node {
    //是否是数据节点（也就标识了是生产者还是消费者）
    final boolean isData;   // false if this is a request node
    //元素
    volatile Object item;   // initially non-null if isData; CASed to match
    //下一个节点
    volatile Node next;
    //持有元素的线程
    volatile Thread waiter; // null until waiting
}
```
### 构造方法

```
    //无参构造方法
    public LinkedTransferQueue() {
    }
    //使用集合构造
    public LinkedTransferQueue(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```
### 入队
四个方法都是一样的，使用异步的方式调用xfer()方法，传入的参数都一模一样。
```
    //插入此队列的尾部指定的元素。 由于队列是无界的，所以此方法不会抛出IllegalStateException或返回false 。
    public boolean add(E e) {
        xfer(e, true, ASYNC, 0);
        return true;
    }
    //插入此队列的尾部指定的元素。 作为队列是无界的，该方法不会阻塞。
    public void put(E e) {
        xfer(e, true, ASYNC, 0);
    }
    //插入此队列的尾部指定的元素。 由于队列是无界的，所以此方法不会返回false 。
    public boolean offer(E e) {
        xfer(e, true, ASYNC, 0);
        return true;
    }
    //插入此队列的尾部指定的元素。timeout和unit不生效
    public boolean offer(E e, long timeout, TimeUnit unit) {
        xfer(e, true, ASYNC, 0);
        return true;
    }
```
### 出队
出队的四个方法也是直接或间接的调用xfer()方法。
```
    public E remove() {
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }
    public E poll() {
        return xfer(null, false, NOW, 0);
    }
    //
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = xfer(null, false, TIMED, unit.toNanos(timeout));
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }
    //阻塞
    public E take() throws InterruptedException {
        E e = xfer(null, false, SYNC, 0);
        if (e != null)
            return e;
        Thread.interrupted();
        throw new InterruptedException();
    }    
```
### tranfer操作

```
    //立即返回
    public boolean tryTransfer(E e) {
        return xfer(e, true, NOW, 0) == null;
    }
    //同步模式
    public void transfer(E e) throws InterruptedException {
        if (xfer(e, true, SYNC, 0) != null) {
            Thread.interrupted(); // failure possible only due to interrupt
            throw new InterruptedException();
        }
    }
    //有超时时间
    public boolean tryTransfer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (xfer(e, true, TIMED, unit.toNanos(timeout)) == null)
            return true;
        if (!Thread.interrupted())
            return false;
        throw new InterruptedException();
    }    
```
### xfer(E e, boolean haveData, int how, long nanos)

```
    //实现所有排队的方法
    //e:元素，如果为null是消费者
    //haveData:为true是生产者，false是消费者
    //how:四种模式NOW, ASYNC, SYNC, or TIMED
    //nanos:超时（以纳秒为单位），仅在模式为TIMED时使用
    private E xfer(E e, boolean haveData, int how, long nanos) {
        if (haveData && (e == null))
            throw new NullPointerException();
        Node s = null;                        // the node to append, if needed

        retry:
        for (;;) {                            // restart on append race
            //队列不为空
            for (Node h = head, p = h; p != null;) { // find & match first node
                //p节点模式
                boolean isData = p.isData;
                //p节点的元素
                Object item = p.item;
                //未被匹配的节点
                if (item != p && (item != null) == isData) { // unmatched
                    //如果两者模式一样，则不能匹配，跳出循环
                    if (isData == haveData)   // can't match
                        break;
                    //进行匹配
                    //把p的值设置为e（如果是消费者则e是null，如果是生产者则e是元素值）
                    if (p.casItem(item, e)) { // match
                        for (Node q = p; q != h;) {
                            Node n = q.next;  // update by 2 unless singleton
                            //如果head还没变，就把它更新成新的节点
                            //更新head为匹配节点的next节点
                            if (head == h && casHead(h, n == null ? q : n)) {
                                //forgetNext()会把它的next设为自己，从单链表中删除
                                h.forgetNext();
                                break;
                            }                 // advance and retry
                            //CAS失败，重新获取head，如果新的头节点为空，或者其next为空，或者其next未匹配，就重试
                            if ((h = head)   == null ||
                                (q = h.next) == null || !q.isMatched())
                                break;        // unless slack < 2
                        }
                        //唤醒p节点等待线程
                        LockSupport.unpark(p.waiter);
                        return LinkedTransferQueue.<E>cast(item);
                    }
                }
                //匹配失败，向后查找节点
                Node n = p.next;
                p = (p != n) ? n : (h = head); // Use head if p offlist
            }
            //未找到匹配节点，把当前节点加入到队列尾
            if (how != NOW) {                 // No matches available
                if (s == null)
                    s = new Node(e, haveData);
                //将新节点s添加到队列尾并返回s的前继节点    
                Node pred = tryAppend(s, haveData);
                if (pred == null)
                    //其他不同模式线程竞争失败重新循环
                    continue retry;           // lost race vs opposite mode
                if (how != ASYNC)
                    //同步操作，等待匹配
                    return awaitMatch(s, pred, e, (how == TIMED), nanos);
            }
            return e; // not waiting
        }
    }
```
1. 从head开始向后匹配，找到一个节点模式跟本次操作的模式不同的未匹配的节点（生产或消费）进行匹配；
2. 匹配节点成功 CAS 修改匹配节点的 item 为给定元素 e；
3. 如果此时所匹配节点向后移动，则 CAS 更新 head 节点为匹配节点的 next 节点，旧 head 节点链接指向自身等待被回收（forgetNext()方法）；如果CAS 失败，并且松弛度大于等于2，就需要重新获取 head 重试。
4. 匹配成功，唤醒匹配节点 p 的等待线程 waiter，返回匹配的 item。
5. 没有找到匹配节点，则根据参数how做不同的处理：
    1. NOW：立即返回;
    2. SYNC：通过tryAppend方法插入一个新的节点 s(item=e,isData = haveData)到队列尾，然后自旋或阻塞当前线程直到节点被匹配或者取消返回;
    3. ASYNC：通过tryAppend方法插入一个新的节点 s(item=e,isData = haveData)到队列尾，异步直接返回;
    4. TIMED：通过tryAppend方法插入一个新的节点 s(item=e,isData = haveData)到队列尾，然后自旋或阻塞当前线程直到节点被匹配或者取消或等待超时返回;

### tryAppend(Node s, boolean haveData)
添加给定节点 s 到队列尾并返回 s 的前继节点，失败时（与其他不同模式线程竞争失败）返回null，没有前继节点返回自身。
```
    private Node tryAppend(Node s, boolean haveData) {
        for (Node t = tail, p = t;;) {        // move p to last node and append
            Node n, u;                        // temps for reads of next & tail
            //如果队列为空，CAS更新s为头节点
            if (p == null && (p = head) == null) {
                //插入第一个元素的时候tail并没有指向s
                if (casHead(null, s))
                    return s;                 // initialize
            }
            else if (p.cannotPrecede(haveData))
                // 如果p无法处理，则返回null
                // 这里无法处理的意思是，p和s节点的类型不一样，不允许s入队
                // 比如，其它线程先入队了一个数据节点，这时候要入队一个非数据节点，就不允许，队列中所有的元素都要保证是同一种类型的节点
                // 返回null后外面的方法会重新尝试匹配重新入队等
                return null;                  // lost race vs opposite mode
            //如果p的next不为空，说明不是最后一个节点
            //则让p重新指向最后一个节点
            else if ((n = p.next) != null)    // not last; keep traversing
                p = p != t && t != (u = tail) ? (t = u) : // stale tail
                    (p != n) ? n : null;      // restart if off list
            // 如果CAS更新s为p的next失败
            // 则说明有其它线程先一步更新到p的next了
            // 就让p指向p的next，重新尝试让s入队
            else if (!p.casNext(null, s))
                p = p.next;                   // re-read on CAS failure
            else {
                // 到这里说明s成功入队了
                // 如果p不等于t，就更新tail指针
                if (p != t) {                 // update if slack now >= 2
                    while ((tail != t || !casTail(t, s)) &&
                           (t = tail)   != null &&
                           (s = t.next) != null && // advance and retry
                           (s = s.next) != null && s != t);
                }
                return p;
            }
        }
    }
```

### awaitMatch(Node s, Node pred, E e, boolean timed, long nanos)
当前操作为同步操作时，会调用awaitMatch方法阻塞等待匹配，成功返回匹配节点 item，失败返回给定参数e（s.item）。在等待期间如果线程被中断或等待超时，则取消匹配，并调用unsplice方法解除节点s和其前继节点的链接。
```
    private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {
        //如果是有超时的，计算其超时时间
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Thread w = Thread.currentThread();
        int spins = -1; // initialized after first item and cancel checks
        // 随机数，随机让一些自旋的线程让出CPU
        ThreadLocalRandom randomYields = null; // bound if needed

        for (;;) {
            Object item = s.item;
            //如果s元素的值不等于e，匹配成功
            if (item != e) {                  // matched
                // assert item != s;
                // 把s的item更新为s本身
                // 并把s中的waiter置为空
                s.forgetContents();           // avoid garbage
                //返回匹配元素
                return LinkedTransferQueue.<E>cast(item);
            }
            //线程中断或有超时的到期了，s元素指向自身
            if ((w.isInterrupted() || (timed && nanos <= 0)) &&
                    s.casItem(e, s)) {        // cancel
                //解除与前一节点联系
                unsplice(pred, s);
                return e;
            }
            //如果自旋次数小于0，就计算自旋次数
            if (spins < 0) {                  // establish spins at/near front
                if ((spins = spinsFor(pred, s.isData)) > 0)
                    //初始化随机数
                    randomYields = ThreadLocalRandom.current();
            }//如果自旋次数大于0，自旋次数-1
            else if (spins > 0) {             // spin
                --spins;
                //随机让出CPU
                if (randomYields.nextInt(CHAINED_SPINS) == 0)
                    Thread.yield();           // occasionally yield
            }//更新s的等待线程为当前线程
            else if (s.waiter == null) {
                s.waiter = w;                 // request unpark then recheck
            }//设置超时
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos > 0L)
                    //阻塞一定时间
                    LockSupport.parkNanos(this, nanos);
            }
            else {
                //直接阻塞，等待被唤醒
                LockSupport.park(this);
            }
        }
    }
```
### 对比
LinkedTransferQueue与SynchronousQueue（公平模式）异同：
1. 在java8中两者的实现方式基本一致，都是使用的双重队列；
2. 前者完全实现了后者，但比后者更灵活；
3. 后者不管放元素还是取元素，如果没有可匹配的元素，所在的线程都会阻塞；
4. 前者可以自己控制放元素是否需要阻塞线程，比如使用四个添加元素的方法就不会阻塞线程，只入队元素，使用transfer()会阻塞线程；
5. 取元素两者基本一样，都会阻塞等待有新的元素进入被匹配到；

## 总结
1. `LinkedTransferQueue`可以看作`LinkedBlockingQueue`、`SynchronousQueue`（公平模式）、`ConcurrentLinkedQueue`三者的集合体；
2. `LinkedTransferQueue`的实现方式是使用一种叫做`双重队列`的数据结构,不管是取元素还是放元素都会入队；
4. 先尝试跟头节点比较，如果二者模式不一样，就匹配它们，然后返回对方的值；
5. 如果二者模式一样，就入队，并自旋或阻塞等待被唤醒；
6. 至于是否入队及阻塞有四种模式，`NOW`、`ASYNC`、`SYNC`、`TIMED`；
7. 对于入队之后，先自旋一定次数后再调用LockSupport.park()或LockSupport.parkNanos阻塞；