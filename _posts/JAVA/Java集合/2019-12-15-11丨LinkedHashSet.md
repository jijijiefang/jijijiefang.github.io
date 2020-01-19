---
layout:     post
title:      "Java集合-11丨LinkedHashSet"
date:       2019-12-15 21:20:57
author:     "jiefang"
header-style: text
tags:
    - Java集合
---
# LinkedHashSet

## 简介
官方注释
><p>Hash table and linked list implementation of the <tt>Set</tt> interface,with predictable iteration order.  This implementation differs from <tt>HashSet</tt> in that it maintains a doubly-linked list running through all of its entries.  This linked list defines the iteration ordering, which is the order in which elements were inserted into the set (<i>insertion-order</i>).  Note that insertion order is <i>not</i> affected if an element is <i>re-inserted</i> into the set.  (An element <tt>e</tt> is reinserted into a set <tt>s</tt> if <tt>s.add(e)</tt> is invoked when <tt>s.contains(e)</tt> would return <tt>true</tt> immediately prior to the invocation.)

翻译
- 哈希表和链接列表实现Set接口，具有可预知的迭代顺序;
- 此实现与HashSet的不同之处在于，后者维护其所有项目上运行的双向链表;
- 此链接列表定义迭代排序，这是在其中元件被插入到该组（ 插入顺序 ）的顺序。 注意，如果一个元件被重新插入到该组插入顺序不受影响 。

LinkedHashSet是一个有序的HashSet。

## 源码
### 构造方法
```
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private static final long serialVersionUID = -2851667679971038690L;
    //指定初始容量和负载因子的构造方法
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }
    //指定初始容量的构造方法
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }
    //无参构造方法
    public LinkedHashSet() {
        super(16, .75f, true);
    }
    //将集合c中的所有元素添加到LinkedHashSet中
    //HashSet中的计算方式是Math.max((int) (c.size()/.75f) + 1, 16)
    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}
```
HashSet的构造方法，使用LinkedHashMap实现，用于LinkedHashSet。
```
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

## 总结
1. LinkedHashSet的底层使用LinkedHashMap存储元素；
2. LinkedHashSet是有序的，它是按照插入的顺序排序的；
3. LinkedHashSet不支持按访问顺序对元素排序的（因为accessOrder是false）；