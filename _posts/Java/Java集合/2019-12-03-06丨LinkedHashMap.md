---
layout:     post
title:      "Java集合-06丨LinkedHashMap"
date:       2019-12-03 22:48:01
author:     "jiefang"
header-style: text
tags:
    - Java集合
---
# LinkedHashMap

## 简介

>LinkedHashMap内部维护了一个双向链表，可以按元素插入顺序访问，可以用来实现LRU缓存策略。<br>
>LinkedHashMap可以看成是 LinkedList + HashMap。

### 类图

![image](https://s2.ax1x.com/2019/12/03/QQA9Vf.png)

### 结构图

![image](https://s2.ax1x.com/2019/12/03/QQAyQA.png)

HashMap 底层的数据结构是：数组 + 链表 + 红黑树。LinkedHashMap继承自HashMap所以它的内部也是这种数据结构。另外它还实现了双向链表的结构存储所有元素。

## 源码

### 属性
```
    //双向链表头结点
    transient LinkedHashMap.Entry<K,V> head;
    //双向链表尾结点
    transient LinkedHashMap.Entry<K,V> tail;
    // true 按照访问顺序，会把经常访问的 key 放到队尾
    // false（默认）按照插入顺序提供访问
    final boolean accessOrder;
```
### 内部类
```
    //LinkedHashMap.Entry
    static class Entry<K,V> extends HashMap.Node<K,V> {
        //维护了元素的前置节点和后继节点
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    //HashMap.
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
```
### 构造方法
不指定`accessOrder`的构造方法默认使用插入顺序存储元素。如果`accessOrder`传入true,则就实现了按访问顺序存储元素，这也是实现LRU缓存策略的关键。
```
    //默认构造方法
    public LinkedHashMap() {
        super();
        //按写入顺序存储元素
        accessOrder = false;
    }
    //指定初始容量构造方法
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }
    //指定初始容量和负载因子构造方法
    public LinkedHashMap(int initialCapacity, float loadFactor){
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }
    //通过指定Map构造
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }
    //指定初始容量、负载因子和排序方式
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }    
```
### afterNodeInsertion(boolean evict)
在节点插入之后做些什么，在HashMap中的putVal()方法中被调用。
```
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```
- 如果为true且头结点不为空且移除最老节点为true，则调用removeNode移除链表头结点。

### afterNodeAccess(Node<K,V> e)
在节点访问之后被调用，主要在put()已经存在的元素或get()时被调用，如果accessOrder为true，把节点移动到双向链表的末尾。
```
    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        //accessOrder为true且尾节点不为空
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            //链表移除p节点
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            //p设置为尾节点
            tail = p;
            ++modCount;
        }
    }
```
1. 如果accessOrder为true且尾节点不为空；
2. 链表移除p节点；
3. p节点设置为尾节点；

### afterNodeRemoval(Node<K,V> e)
HashMap.removeNode方法中调用，删除节点后所做操作。
```
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        //移除e节点
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```
### get(Object key)
返回元素的value或null,如果accessOrder为true,把访问的元素挪至链表尾节点。
```
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```

## 使用
### 实现LRU

```
class LruCache<K,V>  extends LinkedHashMap {
    private final int CACHE_SIZE;

    LruCache(int cacheSize) {
        // true 表示让 linkedHashMap 按照访问顺序来进行排序
        //最近访问的放在头部，最老访问的放在尾部。
        super((int) Math.ceil(cacheSize / 0.75) + 1, 0.75f, true);
        CACHE_SIZE = cacheSize;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        //当 map中的数据量大于指定的缓存个数的时候，就自动删除最老的数据。
        return size() > CACHE_SIZE;
    }
}
```
## 总结
1. LinkedHashMap继承自HashMap，具有HashMap的所有特性；
2. LinkedHashMap内部维护了一个双向链表存储所有的元素；
3. 如果accessOrder为false，按插入元素的顺序遍历元素；
4. 如果accessOrder为true，按访问元素的顺序遍历元素,最近访问元素挪至链表尾节点；
5. LinkedHashMap的实现非常精妙，很多方法都是在HashMap中留的钩子（Hook），直接实现这些Hook就可以实现对应的功能了，并不需要再重写put()等方法；
6. 默认的LinkedHashMap并不会移除旧元素，如果需要移除旧元素，则需要重写removeEldestEntry()方法设定移除策略；
7. LinkedHashMap可以用来实现LRU缓存淘汰策略；