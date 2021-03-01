---
layout:     post
title:      "Java集合-18丨PriorityBlockingQueue"
date:       2019-12-18 14:57:50
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - Java多线程
---
# PriorityBlockingQueue

## 简介
Java注释：
> An unbounded {@linkplain BlockingQueue blocking queue} that uses the same ordering rules as class {@link PriorityQueue} and supplies blocking retrieval operations.  While this queue is logically unbounded,attempted additions may fail due to resource exhaustion (causing {@code OutOfMemoryError}). This class does not permit {@code null} elements.  A priority queue relying on {@linkplain Comparable natural ordering} also does not permit insertion of non-comparable objects (doing so results in {@code ClassCastException}).

翻译：
>一个无界阻塞队列 ，它使用相同的顺序规则类PriorityQueue并提供阻塞检索操作。 尽管此队列在逻辑上是不受限制的，但是尝试尝试添加可能由于资源耗尽而失败（导致{@code OutOfMemoryError}）。 这个类不允许null元素。 依赖于{@linkplain可比自然排序}的优先级队列也不允许插入不可比较的对象（这样做会导致 {@code ClassCastException}）。

`PriorityBlockingQueue`是java并发包下线程安全的优先级阻塞队列。

### 类图
![PriorityBlockingQueue](https://s2.ax1x.com/2019/12/17/QoNtxg.png)

## 源码
### 属性

```java
    //默认初始容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    //最大容量
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //存放元素的数组
    private transient Object[] queue;
    //元素个数
    private transient int size;
    //比较器
    private transient Comparator<? super E> comparator;
    //锁
    private final ReentrantLock lock;
    //条件
    private final Condition notEmpty;
    //扩容的时候使用的控制变量，CAS更新这个值，谁更新成功了谁扩容，其它线程让出CPU
    private transient volatile int allocationSpinLock;
    //不阻塞的优先级队列，非存储元素的地方，仅用于序列化/反序列化时
    private PriorityQueue<E> q;
```
1. 使用数组存储元素；
2. 使用锁和条件保证并发安全；
3. 使用一个变量的CAS操作来控制扩容；

### 构造方法

```java
    //无参构造方法
    public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
    //通过制定集合构造
    public PriorityBlockingQueue(Collection<? extends E> c) {
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        boolean heapify = true; // true if not known to be in heap order
        boolean screen = true;  // true if must screen for nulls
        //如果是SortedSet，按照SortedSet的顺序构造
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            heapify = false;
        }//如果是PriorityBlockingQueue，按照PriorityBlockingQueue顺序构造
        else if (c instanceof PriorityBlockingQueue<?>) {
            PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            screen = false;
            if (pq.getClass() == PriorityBlockingQueue.class) // exact match
                heapify = false;
        }
        Object[] a = c.toArray();
        int n = a.length;
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, n, Object[].class);
        if (screen && (n == 1 || this.comparator != null)) {
            for (int i = 0; i < n; ++i)
                if (a[i] == null)
                    throw new NullPointerException();
        }
        this.queue = a;
        this.size = n;
        if (heapify)
            heapify();
    }
    //指定容量构造
    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }
    //指定容量和比较器构造
    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }    
```
### 入队
#### add(E e)
入队一定成功，不会抛IllegalStateException。
```java
    //调用offer()，不会抛异常
    public boolean add(E e) {
        return offer(e);
    }
```
#### offer(E e)
入队方法，一定成功。
```java
    public boolean offer(E e) {
        //元素null校验
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        int n, cap;
        Object[] array;
        while ((n = size) >= (cap = (array = queue).length))
            //扩容
            tryGrow(array, cap);
        try {
            Comparator<? super E> cmp = comparator;
            //根据是否设置比较器不同处理
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                siftUpUsingComparator(n, e, array, cmp);
            size = n + 1;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
            //父节点
            int parent = (k - 1) >>> 1;
            //父节点的元素e
            Object e = array[parent];
            //如果key比父节点元素e大，结束
            if (key.compareTo((T) e) >= 0)
                break;
            //交换
            array[k] = e;
            k = parent;
        }
        //找到位置，放入
        array[k] = key;
    }    
```
1. 加锁；
2. 判断是否需要扩容；
3. 添加元素并自下而上堆化；
4. 元素个数+1,唤醒notEmpty条件，唤醒取元素阻塞线程；
5. 解锁；

#### put(E e)
入队一定成功，不会阻塞。
```java
    public void put(E e) {
        offer(e); // never need to block
    }
```
#### offer(E e, long timeout, TimeUnit unit)
入队一定成功，不会阻塞一段时间。
```java
    public boolean offer(E e, long timeout, TimeUnit unit) {
        return offer(e); // never need to block
    }
```
### 扩容
#### tryGrow(Object[] array, int oldCap)
```java
    private void tryGrow(Object[] array, int oldCap) {
        //释放锁
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;
        //如果allocationSpinLock是0且CAS修改allocationSpinLock为1成功
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
                //如果旧容量小于64，扩容是2倍+2；否则扩容是1.5倍
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));
                //判断是否达到最大容量
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                //新数组
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }
        //如果新数组为null，当前线程让出CPU
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();
        //再次加锁
        lock.lock();
        //拷贝新数组元素
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```
1. 解锁；
2. CAS修改allocationSpinLock为1；
3. 旧容量小于64则翻倍，旧容量大于64则增加一半；
4. 创建新数组；
5. 修改allocationSpinLock为0；
6. 其它线程在扩容的过程中让出CPU；
7. 再次加锁；
8. 创建新数组，拷贝旧数组元素，返回到offer()方法中继续添加元素操作；

### 出队
#### remove(Object o)
根据元素移除，返回是否成功。
```java
    public boolean remove(Object o) {
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        try {
            //找到元素索引
            int i = indexOf(o);
            if (i == -1)
                return false;
            //移除元素
            removeAt(i);
            return true;
        } finally {
            //解锁
            lock.unlock();
        }
    }
```
#### take()
从队列中取元素，如果队列为空阻塞。
```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        E result;
        try {
            //出队
            while ( (result = dequeue()) == null)
                //队列为空，阻塞
                notEmpty.await();
        } finally {
            //解锁
            lock.unlock();
        }
        //返回
        return result;
    }
```
#### poll()
出队返回元素，可能为null。
```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```
#### poll(long timeout, TimeUnit unit)
出队返回元素，可能为null，队列为空时阻塞一段时间。
```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        //可中断锁
        lock.lockInterruptibly();
        E result;
        try {
            //如果队列为空，超时时长
            while ( (result = dequeue()) == null && nanos > 0)
                //阻塞时间
                nanos = notEmpty.awaitNanos(nanos);
        } finally {
            lock.unlock();
        }
        return result;
    }
```
#### dequeue()
返回数组索引为0的元素。
```java
    private E dequeue() {
        //元素个数-1
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;
            //弹出堆顶元素
            E result = (E) array[0];
            //堆尾元素拿到堆顶
            E x = (E) array[n];
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            //根据比较器不同方法    
            if (cmp == null)
                siftDownComparable(0, x, array, n);
            else
                siftDownUsingComparator(0, x, array, n, cmp);
            //修改size
            size = n;
            return result;
        }
    }
    private static <T> void siftDownComparable(int k, T x, Object[] array,int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            int half = n >>> 1;           // loop while a non-leaf
            //只需要遍历到叶子节点就够了
            while (k < half) {
                //左子节点
                int child = (k << 1) + 1; // assume left child is least
                //左子节点的值
                Object c = array[child];
                //右子节点
                int right = child + 1;
                //取左右子节点中最小的值
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    c = array[child = right];
                //key如果比左右子节点都小，则堆化结束
                if (key.compareTo((T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            //找到了放元素的位置，放置元素
            array[k] = key;
        }
    } 
```
## 总结
1. `PriorityBlockingQueue`整个入队出队的过程与`PriorityQueue`基本是保持一致的；
2. `PriorityBlockingQueue`使用一个锁+一个notEmpty条件控制并发安全；
3. `PriorityBlockingQueue`扩容时使用`allocationSpinLock`的CAS操作来控制只有一个线程进行扩容；
4. 入队使用自下而上的堆化；
5. 出队使用自上而下的堆化；

