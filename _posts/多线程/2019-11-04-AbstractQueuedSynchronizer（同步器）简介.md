---
layout:     post
title:      "AbstractQueuedSynchronizer（同步器）简介"
date:       2019-11-04 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
---
# AbstractQueuedSynchronizer（同步器）简介
## JUC三板斧
了解以下JUC的设计套路，总结三板斧：
>状态，队列，CAS

- **状态**:一般是一个state属性，它基本是整个工具的核心，通常整个工具都是在**设置和修改状态**，很多方法的操作都依赖于当前状态是什么。由于状态是全局共享的，一般会被设置成volatile类型，以保证其修改的可见性；
- **队列**:队列通常是一个等待的集合，大多数以链表的形式实现。队列采用的是悲观锁的思想，表示当前所等待的资源，状态或者条件短时间内可能无法满足。因此，它会将当前线程包装成某种类型的数据结构，扔到一个等待队列中，当一定条件满足后，再从等待队列中取出。
- **CAS**:CAS操作是最轻量的并发处理，通常我们对于状态的修改都会用到CAS操作，因为状态可能被多个线程同时修改，CAS操作保证了同一个时刻，只有一个线程能修改成功，从而保证了线程安全，CAS操作基本是由Unsafe工具类的==compareAndSwapXXX==来实现的；CAS采用的是乐观锁的思想，因此常常伴随着自旋，如果发现当前无法成功地执行CAS，则不断重试，直到成功为止，自旋的的表现形式通常是一个死循环==for(;;)==。

## AQS概述
>AbstractQueuedSynchronizer，简称AQS。是一个用于构建锁和同步器的框架，许多同步器都可以通过AQS很容易并且高效地构造出来，如常用的ReentrantLock、Semaphore、CountDownLatch等。基于AQS来构建同步器能带来许多好处。它不仅能极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题。在基于AQS构建的同步器中，只可能在一个时刻发生阻塞，从而降低上下文切换的开销，并提高吞吐量。

## AQS核心实现

### 状态
AQS中状态是由volatile类型的state属性来表示。
```
private volatile int state;
```
该属性的值即表示了锁的状态，state为0表示锁没有被占用，state大于0表示当前已经有线程持有该锁，这里之所以说大于0而不说等于1是因为可能存在可重入的情况。可以把state变量当做是当前持有该锁的线程数量。

在监视器锁中，ObjectMonitor对象的_owner属性记录了当前拥有监视器锁的线程，而在AQS中，通过==exclusiveOwnerThread==属性表示当前持有锁的线程：
```
//继承自AbstractOwnableSynchronizer
private transient Thread exclusiveOwnerThread;
```
### 队列
AQS的内部实现了两个队列，一个**同步队列**和一个**条件队列**。
- **条件队列**是为Lock实现的一个基础同步器，并且一个线程可能会有多个条件队列，只有在使用了Condition才会存在条件队列。
- **同步队列**的作用是，在线程获取资源失败后，进入同步队列队尾保持自旋等待状态， 在同步队列中的线程在自旋时会判断其前节点是否为head节点，如果为head节点则不断尝试获取资源/锁，获取成功则退出同步队列。当线程执行完逻辑后，会释放资源/锁，释放后唤醒其后继节点。

同步队列和条件队列，都由节点Node组成，它是AQS的内部类：
```
static final class Node {
    //共享模式
    static final Node SHARED = new Node();
    //独占模式
    static final Node EXCLUSIVE = null;
    //取消
    static final int CANCELLED =  1;
    //后续线程需要释放
    static final int SIGNAL    = -1;
    //等待条件
    static final int CONDITION = -2;
    //状态需要向后传播
    static final int PROPAGATE = -3;
    //node节点状态字段,初始化时为0
    volatile int waitStatus;
    //前驱节点
    volatile Node prev;
    //后继节点
    volatile Node next;
    //节点所代表的线程
    volatile Thread thread;
    //独占模式是null，该属性用于条件队列或者共享锁
    Node nextWaiter;
}
```
AQS中双向链表的头节点和尾节点：
```
//头结点，不代表任何线程，是一个哑结点
private transient volatile Node head;
//尾节点，每一个请求锁的线程会加到队尾
private transient volatile Node tail;
```

![image](https://s2.ax1x.com/2019/11/04/KvhVRx.png)



### 四种方法
- **tryAcquire(int arg)**:独占方式。尝试获取资源，成功则返回true，失败则返回false。
- **tryRelease(int arg)**:独占方式。尝试释放资源，成功则返回true，失败则返回false。
- **tryAcquireShared(int arg)**:共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- **tryReleaseShared(int arg)**:共享方式。尝试释放资源，成功则返回true，失败则返回false。