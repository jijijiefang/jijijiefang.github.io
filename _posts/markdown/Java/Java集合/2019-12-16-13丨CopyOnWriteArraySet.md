---
layout:     post
title:      "Java集合-13丨CopyOnWriteArraySet"
date:       2019-12-16 10:34:19
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - Java多线程
---
# CopyOnWriteArraySet

## 简介
官方注释
> A {@link java.util.Set} that uses an internal {@link CopyOnWriteArrayList} for all of its operations.  Thus, it shares the same basic properties:<ul>
    <li>It is best suited for applications in which set sizes generally stay small, read-only operations vastly outnumber mutative operations,and you need to prevent interference among threads during traversal.
    <li>It is thread-safe.
    <li>Mutative operations ({@code add}, {@code set}, {@code remove}, etc.)are expensive since they usually entail copying the entire underlying array.
    <li>Iterators do not support the mutative {@code remove} operation.
    <li>Traversal via iterators is fast and cannot encounter interference from other threads. Iterators rely on unchanging snapshots of the array at the time the iterators were constructed.</ul>

翻译
> 内部使用CopyOnWriteArrayList进行所有操作的Set。因此，它共享相同的基本属性：<ul>
    <li>它最适合于元素个数较少，只读操作远多于可变操作，并且需要防止遍历期间线程间的干扰。
    <li>它是线程安全的。
    <li>可变操作（add，set,remove等）是昂贵的，因为它们通常意味着复制整个底层数组。
    <li>迭代器不支持变化的remove操作。
    <li>通过迭代器遍历速度快，不能从其他线程遇到的干扰。 迭代器依靠在迭代器被构建时不改变所述阵列的快照。</ul>

### 类图

![image](https://s2.ax1x.com/2019/12/16/QhhbFg.png)

## 源码

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    private static final long serialVersionUID = 5457747651344034263L;
    //内部使用CopyOnWriteArrayList存储元素
    private final CopyOnWriteArrayList<E> al;
    //无参构造方法
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
    //指定集合构造方法
    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }
    //元素个数
    public int size() {
        return al.size();
    }
    //是否为空
    public boolean isEmpty() {
        return al.isEmpty();
    }
    //是否包含某元素
    public boolean contains(Object o) {
        return al.contains(o);
    }
    //转化为数组
    public Object[] toArray() {
        return al.toArray();
    }

    
    public <T> T[] toArray(T[] a) {
        return al.toArray(a);
    }
    //清空集合
    public void clear() {
        al.clear();
    }
    //删除
    public boolean remove(Object o) {
        return al.remove(o);
    }
    //添加
    public boolean add(E e) {
        //通过addIfAbsent保证不重复
        return al.addIfAbsent(e);
    }
    //是否包含c中的所有元素
    public boolean containsAll(Collection<?> c) {
        return al.containsAll(c);
    }
    //添加集合
    public boolean addAll(Collection<? extends E> c) {
        return al.addAllAbsent(c) > 0;
    }
    //单方向差集
    public boolean removeAll(Collection<?> c) {
        return al.removeAll(c);
    }
    //交集
    public boolean retainAll(Collection<?> c) {
        return al.retainAll(c);
    }
    //迭代器
    public Iterator<E> iterator() {
        return al.iterator();
    }
    //equals()方法
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Set))
            return false;
        Set<?> set = (Set<?>)(o);
        Iterator<?> it = set.iterator();

        // Uses O(n^2) algorithm that is only appropriate
        // for small sets, which CopyOnWriteArraySets should be.

        //  Use a single snapshot of underlying array
        Object[] elements = al.getArray();
        int len = elements.length;
        // Mark matched elements to avoid re-checking
        boolean[] matched = new boolean[len];
        int k = 0;
        outer: while (it.hasNext()) {
            if (++k > len)
                return false;
            Object x = it.next();
            for (int i = 0; i < len; ++i) {
                if (!matched[i] && eq(x, elements[i])) {
                    matched[i] = true;
                    continue outer;
                }
            }
            return false;
        }
        return k == len;
    }
    //移除满足过滤条件的元素
    public boolean removeIf(Predicate<? super E> filter) {
        return al.removeIf(filter);
    }
    //遍历
    public void forEach(Consumer<? super E> action) {
        al.forEach(action);
    }
    //分割的迭代器
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator
            (al.getArray(), Spliterator.IMMUTABLE | Spliterator.DISTINCT);
    }
    //比较两个元素是否相等
    private static boolean eq(Object o1, Object o2) {
        return (o1 == null) ? o2 == null : o1.equals(o2);
    }
}
```

## 总结
1. `CopyOnWriteArraySet`底层使用`CopyOnWriteArrayList`实现；
2. `CopyOnWriteArraySet`是有序的，因为底层其实是数组；
3. `CopyOnWriteArraySet`是并发安全的，而且实现了读写分离；
4. `CopyOnWriteArraySet`通过调用`CopyOnWriteArrayList`的`addIfAbsent()`方法来保证元素不重复；