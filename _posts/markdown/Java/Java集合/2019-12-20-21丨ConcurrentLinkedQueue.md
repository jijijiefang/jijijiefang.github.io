---
layout:     post
title:      "Java集合-21丨ConcurrentLinkedQueue"
date:       2019-12-20 14:54:31
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - Java多线程
---
# ConcurrentLinkedQueue

## 简介
Java注释
>An unbounded thread-safe {@linkplain Queue queue} based on linked nodes.This queue orders elements FIFO (first-in-first-out).The <em>head</em> of the queue is that element that has been on the queue the longest time.The <em>tail</em> of the queue is that element that has been on the queue the shortest time. New elements are inserted at the tail of the queue, and the queue retrieval operations obtain elements at the head of the queue.A {@code ConcurrentLinkedQueue} is an appropriate choice when many threads will share access to a common collection.Like most other concurrent collection implementations, this class does not permit the use of {@code null} elements.

翻译
>基于链接节点的无界线程安全队列。 此队列对元素FIFO进行排序（先进先出）。 队列的<em> head </ em>是在队列中存在时间最长的元素。队列的<em>tail</em>是队列中最短时间的元素。新元素插入到队列的尾部，而队列检索操作获得位于队列头的元素。当许多线程将共享对一个公共集合的访问权限时，`ConcurrentLinkedQueue`是一个适当的选择。与大多数其他并发集合实现一样，此类不允许使用null元素。

`ConcurrentLinkedQueue`是**单向链表结构的无界并发队列**。从JDK1.7开始加入到J.U.C的行列中。使用CAS实现并发安全，元素操作按照 `FIFO` (first-in-first-out 先入先出) 的顺序。适合“单生产，多消费”的场景。内存一致性遵循对`ConcurrentLinkedQueue`的插入操作先行发生于(happen-before)访问或移除操作。

### 类图

![ConcurrentLinkedQueue](https://s2.ax1x.com/2019/12/20/QO8Uqe.png)

## 源码

### 属性

```java
    //链表头节点
    private transient volatile Node<E> head;
    //链表尾节点
    private transient volatile Node<E> head;
```
### 内部类
```java
private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
}
```
### 构造方法

```java
    //无参构造方法
    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }
    //根据指定集合构造
    public ConcurrentLinkedQueue(Collection<? extends E> c) {
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                //设置头尾节点
                h = t = newNode;
            else {
                //CAS设置头下一节点
                t.lazySetNext(newNode);
                t = newNode;
            }
        }
        if (h == null)
            h = t = new Node<E>(null);
        head = h;
        tail = t;
    }
```
### 入队
因为它不是阻塞队列，所以只有两个入队的方法，add(e)和offer(e)。
```java
    //插入此队列的尾部指定的元素。 由于队列是无界的，所以此方法不会抛出IllegalStateException或返回false 。
    public boolean add(E e) {
        return offer(e);
    }
    //
    public boolean offer(E e) {
        //元素不能为null
        checkNotNull(e);
        //包装成新节点
        final Node<E> newNode = new Node<E>(e);
    
        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            //链表尾部
            if (q == null) {
                // CAS更新p的next为新节点
                // 如果成功了，就返回true
                // 如果不成功就重新取next重新尝试
                if (p.casNext(null, newNode)) {
                    // Successful CAS is the linearization point
                    // for e to become an element of this queue,
                    // and for newNode to become "live".
                    //如果p不等于t，说明有其它线程先一步更新tail
                    if (p != t) // hop two nodes at a time
                        //把tail原子更新为新节点
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                // 如果p的next等于p，说明p已经被删除了（已经出队了）
                // 重新设置p的值
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                //t后面还有值，重新设置p的值
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }    
```
1. 定位到链表尾部，尝试把新节点放到后面；
2. 如果尾部变化了，则重新获取尾部，再重试；

### 出队

因为它不是阻塞队列，所以只有两个出队的方法，remove()和poll()。
```java
    public E remove() {
        E x = poll();
        if (x != null)
            return x;
        else
            throw new NoSuchElementException();
    }
    public E poll() {
        restartFromHead:
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;
                //如果头节点item不为空且CAS更新item为null成功
                if (item != null && p.casItem(item, null)) {
                    // Successful CAS is the linearization point
                    // for item to be removed from this queue.
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                //下面三个分支说明头节点变了
                //如果p的next为空，说明队列中没有元素了
                else if ((q = p.next) == null) {
                    //更新头节点为null
                    updateHead(h, p);
                    return null;
                }//如果p等于p的next，说明p已经出队了，重试
                else if (p == q)
                    continue restartFromHead;
                else//将p设置为p的next
                    p = q;
            }
        }
    }
    //更新链表头节点
    final void updateHead(Node<E> h, Node<E> p) {
        //CAS更新头节点成功后
        if (h != p && casHead(h, p))
            //延迟更新头节点的下一节点指向自己
            h.lazySetNext(h);
    }    
```
1. 定位到头节点，尝试更新其值为null；
2. 如果成功了，就成功出队；
3. 如果失败或者头节点变化了，就重新寻找头节点，并重试；
4. 整个出队过程没有一点阻塞相关的代码，所以出队的时候不会阻塞线程，没找到元素就返回null；

### 不变性

**基本不变性：**

- 当入队插入新节点之后，队列中有一个 next 域为 null （最后一个）的节点。
- 从 head 开始遍历队列，可以访问所有 item 域不为 null 的节点。

**head / tail 的不变性：**

- 所有live节点（指未删除节点），都能从 head 通过调用 succ() 方法遍历可达
- 通过 tail 调用 succ() 方法，最后节点总是可达的。
- head 节点的 next 域不能引用到自身。
- head / tail 不能为 null。

**head / tail 的可变性：**

- head / tail 节点的 item 域可能为 null，也可能不为 null。
- 允许 tail 滞后（lag behind）于 head。也就是说，从 head 开始遍历队列，不一定能到达 tail。
- tail 节点的 next 域可以引用到自身。

## 总结
1. `ConcurrentLinkedQueue`不是阻塞队列；
2. `ConcurrentLinkedQueue`不能用在线程池中；
3. `ConcurrentLinkedQueue`使用（CAS+自旋）更新头尾节点控制出队入队操作；