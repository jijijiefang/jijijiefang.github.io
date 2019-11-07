---
layout:     post
title:      "ThreadLocal原理"
date:       2019-11-05 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
---
# ThreadLocal原理

## 什么是ThreadLocal
ThreadLocal注释：
>This class provides thread-local variables.  These variables differ from
their normal counterparts in that each thread that accesses one (via its
{@code get} or {@code set} method) has its own, independently initialized
copy of the variable.  {@code ThreadLocal} instances are typically private
static fields in classes that wish to associate state with a thread (e.g.,
 a user ID or Transaction ID).
 
 翻译：
 >该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其get 或 set 方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。ThreadLocal实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。
 
 ThreadLocal与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说ThreadLocal为多线程环境下变量问题提供了另外一种解决思路。
 
## 底层原理
在Thread中ThreadLocal.ThreadLocalMap的引用：
```
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```
ThreadLocal.ThreadLocalMap利用Entry来实现key-value的存储：
```
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
}
```

ThreadLocal的get()方法：
```
public T get() {
    //当前线程
    Thread t = Thread.currentThread();
    //从线程里取到ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        //从map里取到当前ThreadLocal的entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            //获取value并返回
            T result = (T)e.value;
            return result;
        }
    }
    //返回默认值
    return setInitialValue();
}
//ThreadLocal.getMap方法
ThreadLocalMap getMap(Thread t) {
    //返回线程的threadLocals属性
    return t.threadLocals;
}

```
ThreadLocal的set()方法：
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

//创建map
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

ThreadLocal的工作原理：
- Thread类中有一个成员变量属于ThreadLocalMap类(一个定义在ThreadLocal类中的内部类)，它是一个Map，他的key是ThreadLocal实例对象。
- 当为ThreadLocal类的对象set值时，首先获得当前线程的ThreadLocalMap类属性，然后以ThreadLocal类的对象为key，设定value。get值时则类似。
- ThreadLocal变量的活动范围为某线程，是该线程“专有的，独自霸占”的，对该变量的所有操作均由该线程完成！也就是说，ThreadLocal 不是用来解决共享对象的多线程访问的竞争问题的，因为ThreadLocal.set() 到线程中的对象是该线程自己使用的对象，其他线程是不需要访问的，也访问不到的。当线程终止后，这些值会作为垃圾回收。

Thread、ThreadLocal、ThreadLocalMap的关系图：

![image](https://s2.ax1x.com/2019/11/06/MP5GNV.png)

**ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用。作用：提供一个线程内公共变量（比如本次请求的用户信息），减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度，或者为线程提供一个私有的变量副本，这样每一个线程都可以随意修改自己的变量副本，而不会对其他线程产生影响。**

## ThreadLocalMap
ThreadLocalMap.getEntry()方法
```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    //快速查询找到hash位置，如果key相等就返回
    if (e != null && e.get() == key)
        return e;
    else
    //循环从下一个位置查找key
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            //移除陈旧数据
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```
ThreadLocalMap.set()方法:
```
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
    int i = key.threadLocalHashCode & (len-1);
    // 采用“线性探测法”，寻找合适位置
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        //key存在，直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }
        // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
        if (k == null) {
            //用新元素替换陈旧的元素
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    //ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //cleanSomeSlots 清除陈旧的Entry（key == null）
    //如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
ThreadLocalMap.remove()方法：
```
//清除key和value防止内存泄露
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            //key设置为null
            e.clear();
            //清除此Entry和其它为null的Entry
            expungeStaleEntry(i);
            return;
        }
    }
}
```
ThreadLocalMap特点：
- 使用ThreadLocal实例对象的**弱引用**作为key；
- key的散列冲突使用**开放定址法**解决，而HashMap的key散列冲突使用**链地址法**解决；
- ThreadLocalMap的初始容量是16，负载因子是2/3。

ThreadLocal对象hashCode的特点：
```
//ThreadLocal一旦创建其散列值就已经确定了，生成过程则是调用nextHashCode()
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode = new AtomicInteger();
//两个ThreadLocal对象hashCode之间差值
private static final int HASH_INCREMENT = 0x61c88647;
//生成下一个hashCode
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```
## ThreadLocal内存泄漏问题
ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用引用他，那么系统gc的时候，这个ThreadLocal势必会被回收，这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：</br>
**Thread Ref** -> **Thread** -> **ThreaLocalMap** -> **Entry** -> **value**</br>
永远无法回收，造成内存泄露。

