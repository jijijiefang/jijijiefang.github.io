---
layout:     post
title:      "Java集合-04丨CopyOnWriteArrayList"
date:       2019-11-29 23:24:11
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - 多线程
---
# CopyOnWriteArrayList
## 简介
Java注释：
>A thread-safe variant of {@link java.util.ArrayList} in which all mutative operations ({@code add}, {@code set}, and so on) are implemented by making a fresh copy of the underlying array.

翻译：
>ArrayList的线程安全变体，其中所有可变的操作（{@code add}，{@ code set}等）都是通过底层数组的新副本实现的。

CopyOnWriteArrayList是ArrayList的线程安全版本，内部也是通过数组实现，每次对数组的修改都完全拷贝一份新的数组来修改，修改完了再替换掉老数组，这样保证了只阻塞写操作，不阻塞读操作，实现读写分离。

### 类图
![CopyOnWriteArrayList类图](https://s2.ax1x.com/2019/11/29/QAYn3Q.png)

## 源码

### 属性

```
    //锁
    final transient ReentrantLock lock = new ReentrantLock();

    //存储元素，只能通过getArray/setArray访问
    private transient volatile Object[] array;
```
使用volatile保证了array的可见性。
### 构造方法

#### CopyOnWriteArrayList()
创建空数组。
```
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }
```
#### CopyOnWriteArrayList(Collection<? extends E> c)
根据传入的集合初始化
```
    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        //如果c是CopyOnWriteArrayList类型，直接赋值
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            //如果不是通过Arrays.copyOf进行拷贝
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }
```
#### CopyOnWriteArrayList(E[] toCopyIn)
根据传入数组初始化
```
    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
```
### 添加
#### add(E e)
添加到列表末尾
```
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取数组
            Object[] elements = getArray();
            int len = elements.length;
            //拷贝生成新数组
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            //赋值
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
1. 加锁；
2. 获取当前数组；
3. 新建一个数组，大小为当前数组加1，拷贝当前数组元素到新数组；
4. 添加元素e到新数组；
5. 新数组设置给array
6. 解锁

#### add(int index, E element)
添加元素到指定位置
```
    public void add(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            //检查数组边界
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
            Object[] newElements;
            int numMoved = len - index;
            //位置在末尾
            if (numMoved == 0)
                newElements = Arrays.copyOf(elements, len + 1);
            else {
                //新数组
                newElements = new Object[len + 1];
                //拷贝旧数组前index的元素到新数组中
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝旧数组index后的元素到新数组中,起始位置为index+1
                System.arraycopy(elements, index, newElements, index + 1,
                                 numMoved);
            }
            //赋值
            newElements[index] = element;
            setArray(newElements);
        } finally {
            lock.unlock();
        }
    }
```
1. 加锁；
2. 检查索引是否越界，index=len属于合法；
3. 如果index=len，拷贝一个len+1的新数组；
4. 如果index!=len,新建len+1数组，拷贝索引之前的部分拷贝到新数组，拷贝索引之后的部分到新数组索引之后，空出索引位置；
5. 元素element设置到索引位置；
6. 新数组设置给array；
7. 解锁；

#### addIfAbsent(E e)
添加一个元素，如果这个元素不存在于集合中。
```
    public boolean addIfAbsent(E e) {
        Object[] snapshot = getArray();
        return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
            addIfAbsent(e, snapshot);
    }
```
如果存在元素，返回元素的索引；否则返回-1。
```
    private static int indexOf(Object o, Object[] elements,
                               int index, int fence) {
        if (o == null) {
            for (int i = index; i < fence; i++)
                if (elements[i] == null)
                    return i;
        } else {
            for (int i = index; i < fence; i++)
                if (o.equals(elements[i]))
                    return i;
        }
        return -1;
    }
```
校验数组是否有修改，添加元素到数组末尾
```
    private boolean addIfAbsent(E e, Object[] snapshot) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //获取旧数组
            Object[] current = getArray();
            int len = current.length;
            //如果快照与刚获取的数组不同，说明有修改
            if (snapshot != current) {
                int common = Math.min(snapshot.length, len);
                for (int i = 0; i < common; i++)
                    //说明元素已经加到最新的数组了，不在快照里
                    if (current[i] != snapshot[i] && eq(e, current[i]))
                        return false;
                if (indexOf(e, current, common, len) >= 0)
                        return false;
            }
            //拷贝新数组
            Object[] newElements = Arrays.copyOf(current, len + 1);
            //元素设置在数组末尾
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
1. 检查元素是否在快照数组中;
2. 存在直接返回false,不存在调用addIfAbsent();
3. 加锁
4. 如果当前数组不等于传入的快照，说明有修改，检查待添加的元素是否存在于当前数组中，如果存在直接返回false;
5. 拷贝新数组；
6. 新元素添加在数组末尾；
7. 新数组复制给array;
8. 解锁；

### 获取
#### get(int index)
获取指定索引的元素，支持随机访问，时间复杂度为O(1)。
```
    public E get(int index) {
        return get(getArray(), index);
    }
    final Object[] getArray() {
        return array;
    }
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
```
1. 获取数组；
2. 获取索引元素；

### 删除
#### remove(int index)
删除指定索引位置的元素。
```
    public E remove(int index) {
        final ReentrantLock lock = this.lock;
        //加锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            //元素位于数组末尾
            if (numMoved == 0)
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                //新数组
                Object[] newElements = new Object[len - 1];
                //拷贝0到index元素
                System.arraycopy(elements, 0, newElements, 0, index);
                //拷贝index到最后的元素
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```
1. 加锁；
2. 获取指定索引旧值；
3. 如果移除的是最后一位元素，则把原数组的前len-1个元素拷贝到新数组中，并把新数组赋值给当前对象的数组属性；
4. 如果移除的不是最后一位元素，则新建一个len-1长度的数组，并把原数组除了指定索引位置的元素全部拷贝到新数组中，并把新数组赋值给当前对象的数组属性；
5. 解锁返回旧值；

### size()
返回数组的长度。
```
    public int size() {
        return getArray().length;
    }
```
## 总结
- CopyOnWriteArrayList使用ReentrantLock重入锁加锁，保证线程安全；
- CopyOnWriteArrayList的写操作都要先拷贝一份新数组，在新数组中做修改，修改完了再用新数组替换老数组，所以空间复杂度是O(n)，性能比较低下；
- CopyOnWriteArrayList的读操作支持随机访问，时间复杂度为O(1)；
- CopyOnWriteArrayList采用读写分离的思想，读操作不加锁，写操作加锁，且写操作占用较大内存空间，所以适用于读多写少的场合；
- CopyOnWriteArrayList只保证最终一致性，不保证实时一致性；