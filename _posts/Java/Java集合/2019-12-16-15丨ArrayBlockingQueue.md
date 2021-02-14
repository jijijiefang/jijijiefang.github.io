---
layout:     post
title:      "Java集合-15丨ArrayBlockingQueue"
date:       2019-12-16 14:03:58
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - Java多线程
---
# ArrayBlockingQueue

## 简介
Java注释
> A bounded {@linkplain BlockingQueue blocking queue} backed by an array.  This queue orders elements FIFO (first-in-first-out).  The
<em>head</em> of the queue is that element that has been on the queue the longest time.  The <em>tail</em> of the queue is that element that has been on the queue the shortest time. New elements are inserted at the tail of the queue, and the queue retrieval operations obtain elements at the head of the queue.

翻译
>由数组实现的有界{@linkplain BlockingQueue阻塞队列}。此队列对元素FIFO（先进先出）进行排序。队列头是队列中最老的元素。队列尾是队列中最新的元素。新元素插入到队列的尾部，而队列检索操作获得位于队列头的元素。

**ArrayBlockingQueue是以数组实现线程安全的阻塞队列。**

### 类图

![ArrayBlockingQueue](https://s2.ax1x.com/2019/12/16/Qh76TP.png)

## 源码

### 属性
```java
    //底层储存元素的数组
    final Object[] items;
    //取元素的索引
    int takeIndex;
    //存放元素的索引
    int putIndex;
    //队列中的元素个数
    int count;
    //主锁
    final ReentrantLock lock;
    //非空条件
    private final Condition notEmpty;
    //非满条件
    private final Condition notFull;
```
1. 底层使用数组存储元素；
2. 通过存索引和取索引标记操作；
3. 通过锁和条件控制并发；

### 构造方法

```java
    //根据指定容量和默认非公平构造
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }
    //根据指定容量和指定公平策略构造
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
    //根据指定容量、指定公平策略、传入集合构造
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);

        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }    
```
1. 初始化必须指定数组的容量；
2. 根据指定公平策略构造可重入锁（默认非公平）；

### 入队
入队有四个方法，它们分别是add(E e)、offer(E e)、put(E e)、offer(E e, long timeout, TimeUnit unit)。

#### add(E e)

```java
    //调用AbstractQueue.add(),调用Queue.offer()
    //其实是ArrayBlockingQueue.offer(E e)
    public boolean add(E e) {
        return super.add(e);
    }
    //AbstractQueue.add()
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    } 
```

#### offer(E e)

```java
    public boolean offer(E e) {
        //元素不能为null
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        try {
            //如果数组已满，返回false
            if (count == items.length)
                return false;
            else {
                //如果数组没满就调用入队方法并返回true
                enqueue(e);
                return true;
            }
        } finally {
            //解锁
            lock.unlock();
        }
    }
```
#### put(E e)

```java
    public void put(E e) throws InterruptedException {
        //元素不能为null
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        try {
            //如果队列已满，notFull阻塞
            while (count == items.length)
                notFull.await();
            //入队
            enqueue(e);
        } finally {
            //解锁
            lock.unlock();
        }
    }
```

#### offer(E e, long timeout, TimeUnit unit)
```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        //元素不能为null
        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        try {
            //如果队列已满，notFull阻塞nanos纳秒
            //如果唤醒这个线程时依然没有空间且时间到了就返回false
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            //入队
            enqueue(e);
            return true;
        } finally {
            //解锁
            lock.unlock();
        }
    }
```

#### enqueue(E x)
私有入队方法，获取锁后使用
```java
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        //如果放指针到数组尽头了，就返回头部
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //notEmpty唤醒阻塞线程
        notEmpty.signal();
    }
```
1. `add(e)`时如果队列满了则抛出异常；
2. `offer(e)`时如果队列满了则返回false；
3. `put(e)`时如果队列满了则使用notFull等待；
4. `offer(e, timeout, unit)`时如果队列满了则等待一段时间后如果队列依然满就返回false；

### 出队
出队有四个方法，它们分别是remove()、poll()、take()、poll(long timeout, TimeUnit unit)

#### remove()

```java
    //AbstractQueue.remove()
    public E remove() {
        //调用poll()方法出队
        E x = poll();
        if (x != null)
            //如果有元素返回
            return x;
        else//没有元素抛异常
            throw new NoSuchElementException();
    }
```
#### poll()

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        try {
            //队列没有元素返回null，否则出队
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```
#### take()

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        try {
            //队列没有元素，notEmpty条件阻塞
            while (count == 0)
                notEmpty.await();
            //出队
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```
#### poll(long timeout, TimeUnit unit)

```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        try {
            //队列没有元素，notEmpty条件阻塞nanos纳秒
            while (count == 0) {
                //如果线程获得了锁但队列依然无元素且已超时就返回null
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            出队
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```
#### dequeue()
私有出队方法，获取锁后调用。
```java
    private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        //取索引前移，如果数组到头了就返回数组前端循环利用
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```
1. `remove()`时如果队列为空则抛出异常；
2. `poll()`时如果队列为空则返回null；
3. `take()`时如果队列为空则阻塞等待在条件`notEmpty`上；
4. `poll(timeout, unit)`时如果队列为空则阻塞等待一段时间后如果还为空就返回null；

## 总结
1. `ArrayBlockingQueue`不需要扩容，初始化时指定容量，并循环利用数组；
2. `ArrayBlockingQueue`利用takeIndex和putIndex循环利用数组；
3. 入队和出队各定义了四组方法为满足不同的用途；
4. 利用重入锁和两个条件保证并发安全；

### ArrayBlockingQueue缺陷
1. 队列长度固定且必须在初始化时指定，所以使用之前一定要慎重考虑好容量；
2. 如果消费速度跟不上入队速度，则会导致生产者线程一直阻塞，且越阻塞越多，非常危险；
3. 只使用了一个锁来控制入队出队，效率较低；

### BlockingQueue方法
BlockingQueue是所有阻塞队列的顶级接口，它里面定义了一批方法.

操作|抛出异常|返回特定值|阻塞|超时
---|---|---|---|---
入队|add(e)|offer(e)——false|put(e)|offer(e, timeout, unit)
出队|remove()|poll()——null|take()|poll(timeout, unit)
检查|element()|peek()——null|-|-

