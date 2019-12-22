---
layout:     post
title:      "Java集合-14丨ConcurrentSkipListSet"
date:       2019-12-22 10:36:47
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - 多线程
---
# ConcurrentSkipListSet

## 简介
Java注释
>A scalable concurrent {@link NavigableSet} implementation based on a {@link ConcurrentSkipListMap}.  The elements of the set are kept sorted according to their {@linkplain Comparable natural ordering},or by a {@link Comparator} provided at set creation time, depending on which constructor is used.

翻译
>基于`ConcurrentSkipListMap`的可伸缩并发`NavigableSet`实现。集合的元素根据它们的可比自然排序或或在集合创建时提供的`Comparator`进行排序，具体取决于使用哪个构造函数。

`ConcurrentSkipListSet`底层是通过`ConcurrentNavigableMap`来实现的，它是一个有序的线程安全的集合。

### 类图

![ConcurrentSkipListSet](https://s2.ax1x.com/2019/12/22/Qzwl38.png)

## 源码
```
public class ConcurrentSkipListSet<E>
    extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {
    //序列化ID
    private static final long serialVersionUID = -2479143111061671589L;
    //使用ConcurrentNavigableMap存储
    private final ConcurrentNavigableMap<E,Object> m;
    //无参构造方法
    public ConcurrentSkipListSet() {
        m = new ConcurrentSkipListMap<E,Object>();
    }
    //指定比较器构造方法
    public ConcurrentSkipListSet(Comparator<? super E> comparator) {
        m = new ConcurrentSkipListMap<E,Object>(comparator);
    }
    //指定集合构造
    public ConcurrentSkipListSet(Collection<? extends E> c) {
        m = new ConcurrentSkipListMap<E,Object>();
        addAll(c);
    }
    //使用SortedSet构造
    public ConcurrentSkipListSet(SortedSet<E> s) {
        m = new ConcurrentSkipListMap<E,Object>(s.comparator());
        addAll(s);
    }
    //使用ConcurrentNavigableMap构造
    ConcurrentSkipListSet(ConcurrentNavigableMap<E,Object> m) {
        this.m = m;
    }
    //克隆
    public ConcurrentSkipListSet<E> clone() {
        try {
            @SuppressWarnings("unchecked")
            ConcurrentSkipListSet<E> clone =
                (ConcurrentSkipListSet<E>) super.clone();
            clone.setMap(new ConcurrentSkipListMap<E,Object>(m));
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new InternalError();
        }
    }

    /* ---------------- Set operations -------------- */
    //元素个数
    public int size() {
        return m.size();
    }
    //集合是否为空
    public boolean isEmpty() {
        return m.isEmpty();
    }
    //是否包含元素o
    public boolean contains(Object o) {
        return m.containsKey(o);
    }
    //添加元素
    public boolean add(E e) {
        return m.putIfAbsent(e, Boolean.TRUE) == null;
    }
    //移除元素
    public boolean remove(Object o) {
        return m.remove(o, Boolean.TRUE);
    }
    //清空集合
    public void clear() {
        m.clear();
    }
    //迭代器
    public Iterator<E> iterator() {
        return m.navigableKeySet().iterator();
    }
    //逆序迭代器
    public Iterator<E> descendingIterator() {
        return m.descendingKeySet().iterator();
    }
    /* ---------------- AbstractSet Overrides -------------- */
    //复写equals方法
    public boolean equals(Object o) {
        // Override AbstractSet version to avoid calling size()
        if (o == this)
            return true;
        if (!(o instanceof Set))
            return false;
        Collection<?> c = (Collection<?>) o;
        try {
            return containsAll(c) && c.containsAll(this);
        } catch (ClassCastException unused) {
            return false;
        } catch (NullPointerException unused) {
            return false;
        }
    }
    //移除c中相等元素
    public boolean removeAll(Collection<?> c) {
        // Override AbstractSet version to avoid unnecessary call to size()
        boolean modified = false;
        for (Object e : c)
            if (remove(e))
                modified = true;
        return modified;
    }

    /* ---------------- Relational operations -------------- */
    //小于e的最大元素
    public E lower(E e) {
        return m.lowerKey(e);
    }
    //小于等于e的最大元素
    public E floor(E e) {
        return m.floorKey(e);
    }
    //大于等于e的最小元素
    public E ceiling(E e) {
        return m.ceilingKey(e);
    }
    //大于e的最小元素
    public E higher(E e) {
        return m.higherKey(e);
    }
    //弹出最小的元素
    public E pollFirst() {
        Map.Entry<E,Object> e = m.pollFirstEntry();
        return (e == null) ? null : e.getKey();
    }
    //弹出最大元素
    public E pollLast() {
        Map.Entry<E,Object> e = m.pollLastEntry();
        return (e == null) ? null : e.getKey();
    }
    /* ---------------- SortedSet operations -------------- */
    //比较器
    public Comparator<? super E> comparator() {
        return m.comparator();
    }
    //返回最小元素
    public E first() {
        return m.firstKey();
    }
    //返回最大元素
    public E last() {
        return m.lastKey();
    }
    //取两个元素之间的子set
    public NavigableSet<E> subSet(E fromElement,
                                  boolean fromInclusive,
                                  E toElement,
                                  boolean toInclusive) {
        return new ConcurrentSkipListSet<E>
            (m.subMap(fromElement, fromInclusive,
                      toElement,   toInclusive));
    }
    //取头子set
    public NavigableSet<E> headSet(E toElement, boolean inclusive) {
        return new ConcurrentSkipListSet<E>(m.headMap(toElement, inclusive));
    }
    //取尾子set
    public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
        return new ConcurrentSkipListSet<E>(m.tailMap(fromElement, inclusive));
    }
    //取子set，包含from，不包含to
    public NavigableSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
    //取头子set，不包含to
    public NavigableSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }
    //取尾子set，包含from
    public NavigableSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }
    //降序set
    public NavigableSet<E> descendingSet() {
        return new ConcurrentSkipListSet<E>(m.descendingMap());
    }
    //分割的迭代器
    @SuppressWarnings("unchecked")
    public Spliterator<E> spliterator() {
        if (m instanceof ConcurrentSkipListMap)
            return ((ConcurrentSkipListMap<E,?>)m).keySpliterator();
        else
            return (Spliterator<E>)((ConcurrentSkipListMap.SubMap<E,?>)m).keyIterator();
    }
    //CAS更新map，给clone方法使用
    private void setMap(ConcurrentNavigableMap<E,Object> map) {
        UNSAFE.putObjectVolatile(this, mapOffset, map);
    }
    //CAS操作相关内容
    private static final sun.misc.Unsafe UNSAFE;
    private static final long mapOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentSkipListSet.class;
            mapOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("m"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}

```
## 总结
1. ConcurrentSkipListSet底层是使用ConcurrentNavigableMap实现的；
2. ConcurrentSkipListSet有序的，基于元素的自然排序或者通过比较器确定的顺序；
3. ConcurrentSkipListSet是线程安全的；


Set|有序性|线程安全|底层实现|关键接口|特点
---|---|---|---|---|---
HashSet|无|否|HashMap|无|简单
LinkedHashSet|有|否|LinkedHashMap|无|插入顺序
TreeSet|有|否|NavigableMap|NavigableSet|自然顺序
CopyOnWriteArraySet|有|是|CopyOnWriteArrayList|无|插入顺序，读写分离
ConcurrentSkipListSet|有|是|ConcurrentNavigableMap|NavigableSet|自然顺序

