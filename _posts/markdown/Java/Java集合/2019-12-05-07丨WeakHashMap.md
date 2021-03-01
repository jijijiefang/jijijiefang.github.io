---
layout:     post
title:      "Java集合-07丨WeakHashMap"
date:       2019-12-05 23:15:53
author:     "jiefang"
header-style: text
tags:
    - Java集合
---
# WeakHashMap
## 简介

> Hash table based implementation of the <tt>Map</tt> interface, with <em>weak keys</em>.

`WeakHashMap`是使用弱引用作为key来实现Map的散列表。当JVM GC时，如果key没有强引用存在时，下次操作会把对应Entry整个删除。

## 原理
`WeakHashMap`的存储结构是（数组 + 链表）来实现。

## 源码

### 属性
```java
    //默认初始容量
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    //最大容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认装载因子
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //Entry数组
    Entry<K,V>[] table;
    //元素个数
    private int size;
    //扩容阈值capacity * loadFactor
    private int threshold;
    //装载因子
    private final float loadFactor;
    //引用队列，当弱键失效的时候会把Entry添加到这个队列中
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
```
### 内部类

#### Entry
`WeakHashMap`内部的存储节点, 没有key属性。

```java
    private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
        V value;
        final int hash;
        Entry<K,V> next;

        //构造方法
        Entry(Object key, V value,
              ReferenceQueue<Object> queue,
              int hash, Entry<K,V> next) {
            //调用WeakReference的构造方法初始化key和引用队列
            super(key, queue);
            this.value = value;
            this.hash  = hash;
            this.next  = next;
        }
        
        @SuppressWarnings("unchecked")
        public K getKey() {
            return (K) WeakHashMap.unmaskNull(get());
        }

        public V getValue() {
            return value;
        }

        public V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            K k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                V v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))
                    return true;
            }
            return false;
        }

        public int hashCode() {
            K k = getKey();
            V v = getValue();
            return Objects.hashCode(k) ^ Objects.hashCode(v);
        }

        public String toString() {
            return getKey() + "=" + getValue();
        }
    }
```
### 构造函数

```java
    //无参构造方法
    public WeakHashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
    }
    //指定初始容量构造方法
    public WeakHashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    //指定初始容量和负载因子构造方法
    public WeakHashMap(int initialCapacity, float loadFactor) {
        //最小边界判断
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Initial Capacity: "+initialCapacity);
        //最大边界判断
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load factor: "+loadFactor);
        int capacity = 1;
        //小于默认就两倍扩容
        while (capacity < initialCapacity)
            capacity <<= 1;
        table = newTable(capacity);
        this.loadFactor = loadFactor;
        //扩容阈值
        threshold = (int)(capacity * loadFactor);
    }    
```
### put(K key, V value)
添加元素。
```java
    public V put(K key, V value) {
        //如果key为null，则新建一个Object对象作为key
        Object k = maskNull(key);
        int h = hash(k);
        Entry<K,V>[] tab = getTable();
        int i = indexFor(h, tab.length);
        //如果在数组上或者链表上找到key相等，则更新value
        for (Entry<K,V> e = tab[i]; e != null; e = e.next) {
            if (h == e.hash && eq(k, e.get())) {
                V oldValue = e.value;
                if (value != oldValue)
                    e.value = value;
                return oldValue;
            }
        }

        modCount++;
        Entry<K,V> e = tab[i];
        //没找到则新增元素，设置在数组上
        tab[i] = new Entry<>(k, value, queue, h, e);
        //判断是否需要扩容
        if (++size >= threshold)
            resize(tab.length * 2);
        return null;
    }
```
1. 如果key为null，则新建一个Object对象作为key；
2. 计算hash值（与HashMap的方式不同）；
3. 计算数组下标；
4. 在数组或者数组对应链表中遍历；
5. 如果找到元素替换成新的value，返回旧的value；
6. 没找到元素则在数组对应位置新增元素；
7. 新增后，如果元素数量>=扩容阈值，2倍扩容（HashMap中是大于threshold才扩容）；

### resize(int newCapacity)
扩容方法。
```java
    void resize(int newCapacity) {
        Entry<K,V>[] oldTable = getTable();
        int oldCapacity = oldTable.length;
        //达到最大容量不在扩容，返回
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        
        Entry<K,V>[] newTable = newTable(newCapacity);
        //旧数组+链表中的数据挪到新的里去
        transfer(oldTable, newTable);
        table = newTable;
        //如果元素个数大于扩容阈值一半，则使用新数组和新容量，并计算新的扩容阈值
        if (size >= threshold / 2) {
            threshold = (int)(newCapacity * loadFactor);
        } else {
            //否则把元素再转移回旧桶，还是使用旧桶
            //因为在transfer的时候会清除失效的Entry，所以元素个数可能没有那么大了，就不需要扩容了
            expungeStaleEntries();
            transfer(newTable, oldTable);
            table = oldTable;
        }
    }
    private void transfer(Entry<K,V>[] src, Entry<K,V>[] dest){
        for (int j = 0; j < src.length; ++j) {
            Entry<K,V> e = src[j];
            src[j] = null;
            while (e != null) {
                Entry<K,V> next = e.next;
                Object key = e.get();
                //如果key为null,则移除元素
                if (key == null) {
                    e.next = null;  // Help GC
                    e.value = null; //  "   "
                    size--;
                //把数组和链表上的元素挪至新的
                } else {
                    int i = indexFor(e.hash, dest.length);
                    e.next = dest[i];
                    dest[i] = e;
                }
                e = next;
            }
        }
    }
```
1. 判断旧容量是否达到最大容量；
2. 新建新数组把旧元素挪至新的数组里；
3. 如果转移后元素个数不到扩容门槛的一半，则把元素再转移回旧桶，继续使用旧桶，说明不需要扩容；
4. 否则使用新数组，并计算新的扩容门槛；
5. 转移元素的过程中会把key为null的元素清除掉，所以size会变小；

### get(Object key)
获取元素。
```java
    public V get(Object key) {
        Object k = maskNull(key);
        int h = hash(k);
        Entry<K,V>[] tab = getTable();
        int index = indexFor(h, tab.length);
        Entry<K,V> e = tab[index];
        while (e != null) {
            if (e.hash == h && eq(k, e.get()))
                return e.value;
            e = e.next;
        }
        return null;
    }
```
1. 如果key为null，则新建一个Object对象作为key；
2. 计算hash值；
3. 找到数组索引；
4. 循环判断数组索引元素或链表中是否存在key相等；
5. 存在则返回value，否则返回null;

### remove(Object key)
删除元素
```java
    public V remove(Object key) {
        Object k = maskNull(key);
        int h = hash(k);
        Entry<K,V>[] tab = getTable();
        int i = indexFor(h, tab.length);
        Entry<K,V> prev = tab[i];
        Entry<K,V> e = prev;

        while (e != null) {
            Entry<K,V> next = e.next;
            if (h == e.hash && eq(k, e.get())) {
                modCount++;
                size--;
                if (prev == e)
                    tab[i] = next;
                else
                    prev.next = next;
                return e.value;
            }
            prev = e;
            e = next;
        }

        return null;
    }
```
1. 如果key为null，则新建一个Object对象作为key；
2. 2. 计算hash值；
3. 找到数组索引；
4. 循环判断数组索引元素或链表中是否存在key相等；
5. 存在则移除元素，并返回value;
6. 否则返回null;

### expungeStaleEntries()
移除失效元素
```java
    private void expungeStaleEntries() {
        //遍历引用队列
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);
                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        //找到该元素
                        if (prev == e)
                            //删除
                            table[i] = next;
                        else
                            prev.next = next;
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
    }
```
1. 当key失效的时候gc会自动把对应的Entry添加到这个引用队列中；
2. 所有对map的操作都会直接或间接地调用到这个方法先移除失效的Entry，比如getTable()、size()、resize()；
3. 这个方法就是遍历引用队列，并把其中保存的Entry从map中移除掉；
4. 移除Entry的同时把value也置为null帮助gc清理；

## 示例


## 总结
1. `WeakHashMap`使用数组+链表的数据结构；
2. `WeakHashMap`的key是弱引用，在GC时会被清除；
3. 每次操作`WeakHashMap`，都会清除失效的key对应的Entry;
4. 使用String作为key时，一定要使用new String()这样的方式声明key，才会失效，其它的基本类型的包装类型是一样的；
5. `WeakHashMap`常用来作为缓存使用；