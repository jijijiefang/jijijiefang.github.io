---
layout:     post
title:      "Java集合-16丨LinkedBlockingQueue"
date:       2019-12-16 15:28:18
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - 多线程
---
# LinkedBlockingQueue

## 简介
Java注释
> An optionally-bounded {@linkplain BlockingQueue blocking queue} based on linked nodes.
This queue orders elements FIFO (first-in-first-out).
The <em>head</em> of the queue is that element that has been on the
queue the longest time.
The <em>tail</em> of the queue is that element that has been on the queue the shortest time. New elements are inserted at the tail of the queue, and the queue retrieval operations obtain elements at the head of the queue.Linked queues typically have higher throughput than array-based queues but less predictable performance in most concurrent applications.
 
翻译
> 可选地具有有界阻塞队列基于链接节点。 这个队列的命令元素FIFO（先入先出）。 队列的头是元素一直在队列中时间最长。 队列的尾部是该元素已经在队列中的时间最短。 新元素插入到队列的尾部，并且队列检索操作获取在队列的头部元素。 链接队列通常比基于阵列的队列，但在大多数并发应用程序少可预测的性能更高的吞吐量。

**LinkedBlockingQueue是基于单链表实现的线程安全的阻塞队列**。

### 类图

![LinkedBlockingQueue](https://s2.ax1x.com/2019/12/16/Q4mpkt.png)

## 源码

### 属性

```
    //容量
    private final int capacity;
    //元素数量
    private final AtomicInteger count = new AtomicInteger();
    //链表头
    transient Node<E> head;
    //链表尾
    private transient Node<E> last;
    //take锁
    private final ReentrantLock takeLock = new ReentrantLock();
    //notEmpty条件，当队列无元素时，take锁会阻塞在notEmpty条件上，等待其它线程唤醒
    private final Condition notEmpty = takeLock.newCondition();
    //put锁
    private final ReentrantLock putLock = new ReentrantLock();
    //notFull条件，当队列满了时，put锁会会阻塞在notFull上，等待其它线程唤醒
    private final Condition notFull = putLock.newCondition();
```
1. capacity，有容量，LinkedBlockingQueue是有界队列()；
2. 入队、出队使用两个不同的锁控制，锁分离，提高效率；

### 内部类
单链表结构
```
    static class Node<E> {
        E item;
        Node<E> next;
        Node(E x) { item = x; }
    }
```
### 构造方法

```
    //无参构造方法
    public LinkedBlockingQueue() {
        //使用最大int数作为容量
        this(Integer.MAX_VALUE);
    }
    //指定容量构造方法
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
    //指定集合构造方法
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```
### 入队
#### add(E e)
添加元素成功返回true,队列满抛异常。
```
    //AbstractQueue.add
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```
#### offer(E e)
添加元素，队列满，返回false。
```
    public boolean offer(E e) {
        //非空判断
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        //队列满，返回false
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        //putLock加锁
        putLock.lock();
        try {
            //队列没满
            if (count.get() < capacity) {
                //入队
                enqueue(node);
                //元素个数加1
                c = count.getAndIncrement();
                //队列没满，唤醒一个阻塞在notFull条件上的线程
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            //释放
            putLock.unlock();
        }
        //如果原队列长度为0，现在加了一个元素后立即唤醒notEmpty条件
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }
```
#### put(E e)
添加元素，队列满，阻塞直到入队或被中断。
```
    public void put(E e) throws InterruptedException {
        //非空判断
        if (e == null) throw new NullPointerException();
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        //可中断锁
        putLock.lockInterruptibly();
        try {
            //如果元素个数等于容量，队列已满
            while (count.get() == capacity) {
                //阻塞
                notFull.await();
            }
            //入队
            enqueue(node);
            //元素个数加1
            c = count.getAndIncrement();
            //队列没满，唤醒一个阻塞在notFull条件上的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        //如果原队列长度为0，现在加了一个元素后立即唤醒notEmpty条件
        if (c == 0)
            signalNotEmpty();
    }
```
#### offer(E e, long timeout, TimeUnit unit)

```
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        //非空校验
        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        //可中断锁
        putLock.lockInterruptibly();
        try {
            //队列满，阻塞nanos纳秒
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            //入队
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            //队列没满，唤醒一个阻塞在notFull条件上的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        //如果原队列长度为0，现在加了一个元素后立即唤醒notEmpty条件
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

#### enqueue(Node<E> node)
入队操作
```
    private void enqueue(Node<E> node) {
        last = last.next = node;
    }
```

#### signalNotEmpty()
唤醒等待线程。
```
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }
```
### 出队

#### remove()
队列移除元素，移除成功返回true，否则抛异常。
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
出队，如果队列为空返回null，否则返回出队元素。
```
    public E poll() {
        final AtomicInteger count = this.count;
        //队列为空，返回null
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        //takeLock加锁
        takeLock.lock();
        try {
            if (count.get() > 0) {
                //出队
                x = dequeue();
                //元素个数减1
                c = count.getAndDecrement();
                //队列非空，唤醒阻塞在notEmpty上的线程
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        //如果队列满，唤醒notFull阻塞线程
        if (c == capacity)
            signalNotFull();
        return x;
    }
```
#### take()
出队，如果队列为空，阻塞，直到队列添加元素。
```
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        //可中断锁
        takeLock.lockInterruptibly();
        try {
            //队列为空
            while (count.get() == 0) {
                //阻塞在notEmpty条件
                notEmpty.await();
            }
            //出队
            x = dequeue();
            //取之前元素个数并减1
            c = count.getAndDecrement();
            //之前元素个数大于1，唤醒阻塞在notEmpty条件线程
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        //之前队列已满
        if (c == capacity)
            signalNotFull();
        return x;
    }
```
#### poll(long timeout, TimeUnit unit)
出队，如果队列为空，阻塞timeout纳秒，直到超时返回null。
```
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E x = null;
        int c = -1;
        long nanos = unit.toNanos(timeout);
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        //可中断锁
        takeLock.lockInterruptibly();
        try {
            //队列为空，阻塞nanos纳秒
            while (count.get() == 0) {
                //超时返回null
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            //出队
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```


## 总结
1. `LinkedBlockingQueue`底层使用单链表实现；
2. `LinkedBlockingQueue`采用两把锁分离技术实现入队出队互不阻塞；
3. `LinkedBlockingQueue`是有界队列，不穿入容量默认为最大int值;

### 对比
1. `ArrayBlockingQueue`入队出队采用一把锁，导致入队出队相互阻塞，效率低下；
2. `LinkedBlockingQueue`入队出队采用两把锁，入队出队互不干扰，效率较高；
3. `LinkedBlockingQueue`和`ArrayBlockingQueue`都是有界队列，如果长度相等且出队速度跟不上入队速度，都会导致大量线程阻塞；
4. `LinkedBlockingQueue`如果初始化不传入初始容量，则使用最大int值，如果出队速度跟不上入队速度，会导致队列特别长，占用大量内存；