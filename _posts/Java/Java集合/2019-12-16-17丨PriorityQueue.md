---
layout:     post
title:      "Java集合-17丨PriorityQueue"
date:       2019-12-16 18:04:58
author:     "jiefang"
header-style: text
tags:
    - Java集合
---
# PriorityQueue

## 简介
Java注释
> An unbounded priority {@linkplain Queue queue} based on a priority heap.The elements of the priority queue are ordered according to their {@linkplain Comparable natural ordering}, or by a {@link Comparator}
provided at queue construction time, depending on which constructor is used.  A priority queue does not permit {@code null} elements.A priority queue relying on natural ordering also does not permit insertion of non-comparable objects (doing so may result in {@code ClassCastException}).

翻译
>基于优先级堆的无限优先级{@linkplain Queue队列}。优先级队列的元素根据它们的{@linkplain可比自然排序}或通过在队列构建时提供的{@link Comparator}进行排序，具体取决于使用的是哪个构造函数。优先级队列不允许{@code null}元素。依赖自然顺序的优先级队列也不允许插入不可比较的对象（这样做可能会导致{@code ClassCastException}）。

**优先级队列，是0个或多个元素的集合，集合中的每个元素都有一个权重值，每次出队都弹出优先级最大或最小的元素，一般来说，优先级队列使用堆来实现**。
### 类图

![PriorityQueue](https://s2.ax1x.com/2019/12/16/Q4rQds.png)

## 源码

### 属性

```java
    //默认容量
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    //储存元素的地方
    transient Object[] queue;
    //元素个数
    private int size = 0;
    //比较器
    private final Comparator<? super E> comparator;
    //修改次数
    transient int modCount = 0;
```
1. 默认容量是11；
2. queue，元素存储在数组中；
3. comparator，比较器，在优先级队列中，也有两种方式比较元素，一种是元素的自然顺序，一种是通过比较器来比较；
4. modCount，修改次数，有这个属性表示PriorityQueue是fast-fail的；

### 构造方法
```java
    //默认容量，自然排序构造
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }
    //指定容量，自然排序构造
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }
    //默认容量，指定排序构造
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }
    //指定容量，指定排序构造
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
    
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }
    //指定集合构造
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }
    //通过指定PriorityQueue构造
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }
    //通过有序集合构造
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
    //通过指定PriorityQueue初始化
    private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
        if (c.getClass() == PriorityQueue.class) {
            this.queue = c.toArray();
            this.size = c.size();
        } else {
            initFromCollection(c);
        }
    }
    //通过普通集合初始化
    private void initElementsFromCollection(Collection<? extends E> c) {
        Object[] a = c.toArray();
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, a.length, Object[].class);
        int len = a.length;
        if (len == 1 || this.comparator != null)
            for (int i = 0; i < len; i++)
                if (a[i] == null)
                    throw new NullPointerException();
        this.queue = a;
        this.size = a.length;
    }    
```
### 入队
入队有两个方法，add(E e)和offer(E e)，两者是一致的，add(E e)也是调用的offer(E e)。

```java
    public boolean add(E e) {
        return offer(e);
    }
    public boolean offer(E e) {
        //非空校验
        if (e == null)
            throw new NullPointerException();
        //修改次数+1
        modCount++;
        int i = size;
        //元素个数达到最大容量，扩容
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        //空队列，放在队列第一个位置
        if (i == 0)
            queue[0] = e;
        else//否则插在数组size位置，堆化
            siftUp(i, e);
        return true;
    }
    //根据是否有比较器，调用不同的方法
    private void siftUp(int k, E x) {
        if (comparator != null)
            siftUpUsingComparator(k, x);
        else
            siftUpComparable(k, x);
    }
    //
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            // 找到父节点的位置
            // 因为元素是从0开始的，所以减1之后再除以2
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            // 比较插入的元素与父节点的值
            // 如果比父节点大，则跳出循环，否则交换位置
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            // 现在插入的元素位置移到了父节点的位置
            // 继续与父节点再比较
            k = parent;
        }
        //最后找到应该插入的位置，放入元素
        queue[k] = key;
    }
    //使用指定比较器比较
    private void siftUpUsingComparator(int k, E x) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (comparator.compare(x, (E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = x;
    }    
```
1. 入队不允许null元素；
2. 如果数组不够用，扩容；
3. 如果队列没有元素，插入下标0位置；
4. 如果有元素了，就插入到最后一个元素往后的一个位置；
5. 自下而上堆化，一直往上跟父节点比较；
6. 如果比父节点小，就与父节点交换位置，直到出现比父节点大为止；
7. PriorityQueue是一个小顶堆；

### 扩容

```java
    private void grow(int minCapacity) {
        int oldCapacity = queue.length;
        // Double size if small; else grow by 50%
        // 旧容量小于64时，容量翻倍
        // 旧容量大于等于64时，增加50%
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
        //是否超过最大值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        queue = Arrays.copyOf(queue, newCapacity);
    }
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }    
```
### 出队
出队有两个方法，remove()和poll()，remove()也是调用的poll()，只是没有元素的时候抛出异常。
```java
    //AbstractQueue.remove()
    public E remove() {
        //调用poll弹出队首元素
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }
    public E poll() {
        //队列中没有元素返回null
        if (size == 0)
            return null;
        //
        int s = --size;
        //修改次数+1
        modCount++;
        //索引0位置元素取出
        E result = (E) queue[0];
        //队列末尾元素
        E x = (E) queue[s];
        //队列末尾设置为null
        queue[s] = null;
        if (s != 0)
            //将队列末元素移到队列首，自上而下堆化
            siftDown(0, x);
        return result;
    }
    //根据是否设置比较器，选择不同的方法
    private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        //只需要比较一半就行了，因为叶子节点占了一半的元素
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            //寻找子节点的位置，这里加1是因为元素从0号位置开始
            int child = (k << 1) + 1; // assume left child is least
            // 左子节点的值
            Object c = queue[child];
            // 右子节点的位置
            int right = child + 1;
            //元素E比右字节点大
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                //左右节点取其小者
                c = queue[child = right];
            // 如果比子节点都小，则结束
            if (key.compareTo((E) c) <= 0)
                break;
            //如果比最小的子节点大，则交换位置
            queue[k] = c;
            //指针移到最小子节点的位置继续往下比较
            k = child;
        }
        //找到正确的位置，放入元素
        queue[k] = key;
    }    
```
### 取队首元素
队首元素有两个方法，element()和peek()，element()也是调用的peek()，只是没取到元素时抛出异常。

```java
    //AbstractQueue.element()
    public E element() {
        E x = peek();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }
    public E peek() {
        return (size == 0) ? null : (E) queue[0];
    }    
```

## 总结
1. `PriorityQueue`是一个小顶堆；
2. `PriorityQueue`是非线程安全的；
3. `PriorityQueue`不是有序的，只有堆顶存储着最小的元素；
4. 入队就是堆的插入元素的实现；
5. 出队就是堆的删除元素的实现；

### Queue方法
Queue是所有队列的顶级接口，它里面定义了一批方法.

操作|抛出异常|返回特定值
---|---|---
入队 |add(e)|offer(e)——false
出队 |remove()|poll()——null
检查 |element()|peek()——null
