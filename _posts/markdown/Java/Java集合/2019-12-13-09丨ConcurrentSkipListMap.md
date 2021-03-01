---
layout:     post
title:      "Java集合-09丨ConcurrentSkipListMap"
date:       2019-12-13 22:30:26
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
    - Java多线程
---
# ConcurrentSkipListMap

## 简介
- 跳表是一个随机化的数据结构，实质就是一种可以进行二分查找的有序链表。
- 跳表在原有的有序链表上面增加了多级索引，通过索引来实现快速查找。
- 跳表不仅能提高搜索性能，同时也可以提高插入和删除操作的性能。

### 存储结构

![跳表](https://s2.ax1x.com/2019/12/13/Q2ktKK.png)

### 类图

![类图](https://s2.ax1x.com/2019/12/13/Q2Z6YQ.png)

`ConcurrentSkipListMap`是线程安全的有序的哈希表，适用于高并发的场景。

## 源码

### 内部类

- Node，数据节点，存储数据的节点，典型的单链表结构；
- Index，索引节点，存储着对应的node值，及向下和向右的索引指针；
- HeadIndex，头索引节点，继承自Index，并扩展level字段，用于记录索引的层级；

#### Node<K,V>

```java
    //数据节点，单向链表结构
    static final class Node<K,V> {
        final K key;
        //value的类型是Object，而不是V
        volatile Object value;
        volatile Node<K,V> next;
        //常规节点
        Node(K key, Object value, Node<K,V> next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
        //标记节点
        Node(Node<K,V> next) {
            this.key = null;
            this.value = this;
            this.next = next;
        }
    }
```
#### Index<K,V>
```java
    //索引节点
    static class Index<K,V> {
        final Node<K,V> node;
        final Index<K,V> down;
        volatile Index<K,V> right;

        Index(Node<K,V> node, Index<K,V> down, Index<K,V> right) {
            this.node = node;
            this.down = down;
            this.right = right;
        }
    }
```

#### HeadIndex<K,V>

```java
    //头索引节点，继承自Index，并扩展level字段，用于记录索引的层级
    static final class HeadIndex<K,V> extends Index<K,V> {
        final int level;
        HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
            super(node, down, right);
            this.level = level;
        }
    }
```

### 构造方法

```java
    //自然排序
    public ConcurrentSkipListMap() {
        this.comparator = null;
        initialize();
    }
    //根据指定的比较器进行排序
    public ConcurrentSkipListMap(Comparator<? super K> comparator){
        this.comparator = comparator;
        initialize();
    }
    //通过指定Map构造，自然排序
    public ConcurrentSkipListMap(Map<? extends K, ? extends V> m) {
        this.comparator = null;
        initialize();
        putAll(m);
    }
    //通过指定有序Map构造
    public ConcurrentSkipListMap(SortedMap<K, ? extends V> m) {
        this.comparator = m.comparator();
        initialize();
        buildFromSorted(m);
    }    
    
```
#### initialize()
初始化方法，构造头索引节点，层级1，down和right都是null。
```java
    private void initialize() {
        keySet = null;
        entrySet = null;
        values = null;
        descendingMap = null;
        head = new HeadIndex<K,V>(new Node<K,V>(null, BASE_HEADER, null),null, null, 1);
    }
```
### 添加

#### put(K key, V value)
```java
    public V put(K key, V value) {
        //空value校验
        if (value == null)
            throw new NullPointerException();
        return doPut(key, value, false);
    }
```
#### doPut(K key, V value, boolean onlyIfAbsent)
```java
    private V doPut(K key, V value, boolean onlyIfAbsent) {
        Node<K,V> z;             // added node
        //key不能为null
        if (key == null)
            throw new NullPointerException();
        //比机器
        Comparator<? super K> cmp = comparator;
        //找到目标节点的位置并插入
        //这里的目标节点是数据节点，也就是最底层的那条链
        outer: for (;;) {
            //寻找目标节点之前最近的一个索引对应的数据节点，存储在b中，b=before
            //并把b的下一个数据节点存储在n中，n=next
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                //如果下一个节点不为空
                //就拿其key与目标节点的key比较，找到目标节点应该插入的位置
                if (n != null) {
                    Object v; int c;
                    Node<K,V> f = n.next;
                    //节点已变化，跳出
                    if (n != b.next)               // inconsistent read
                        break;
                    //value为空，协助删除节点
                    if ((v = n.value) == null) {   // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    //如果b的值为空或者v等于n，说明b已被删除
                    //n就是marker节点，那b就是被删除的那个
                    if (b.value == null || v == n) // b is deleted
                        break;
                    //如果目标key与下一个节点的key大
                    //说明目标元素所在的位置还在下一个节点的后面
                    if ((c = cpr(cmp, key, n.key)) > 0) {
                        // 就把当前节点往后移一位
                        // 同样的下一个节点也往后移一位
                        // 再重新检查新n是否为空，它与目标key的关系
                        b = n;
                        n = f;
                        continue;
                    }
                    //下一个节点的key与目标key相同
                    if (c == 0) {
                        //替换旧值，并返回
                        if (onlyIfAbsent || n.casValue(v, value)) {
                            @SuppressWarnings("unchecked") V vv = (V)v;
                            return vv;
                        }
                        break; // restart if lost race to replace value
                    }
                    //如果c<0，就往下走，也就是找到了目标节点的位置
                    // else c < 0; fall through
                }
                // 有两种情况会到这里
                // 一是到链表尾部了，也就是n为null了
                // 二是找到了目标节点的位置，也就是上面的c<0
                // 新建目标节点，并赋值给z
                z = new Node<K,V>(key, value, n);
                //CAS更新b的下一个节点为目标节点z
                if (!b.casNext(n, z))
                    //失败，跳出
                    break;         // restart if lost race to append to b
                //更新成功，跳出自旋
                break outer;
            }
        }
    
        //随机决定是否需要建立索引及其层次，如果需要则建立自上而下的索引
        int rnd = ThreadLocalRandom.nextSecondarySeed();
        // 0x80000001展开为二进制为10000000000000000000000000000001
        // 只有两头是1
        // 这里(rnd & 0x80000001) == 0
        // 相当于排除了负数（负数最高位是1），排除了奇数（奇数最低位是1）
        // 只有最高位最低位都不为1的数跟0x80000001做&操作才会为0
        // 也就是正偶数
        if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
            //默认level为1，也就是只要到这里了就会至少建立一层索引
            int level = 1, max;
            // 随机数从最低位的第二位开始，有几个连续的1则level就加几
            // 因为最低位肯定是0，正偶数嘛
            // 比如，1100110，level就加2
            while (((rnd >>>= 1) & 1) != 0)
                ++level;
            // 用于记录目标节点建立的最高的那层索引节点
            Index<K,V> idx = null;
            // 取头索引节点（这是最高层的头索引节点）
            HeadIndex<K,V> h = head;
            // 如果生成的层数小于等于当前最高层的层级
            // 也就是跳表的高度不会超过现有高度            
            if (level <= (max = h.level)) {
                // 从第一层开始建立一条竖直的索引链表
                // 这条链表使用down指针连接起来
                // 每个索引节点里面都存储着目标节点这个数据节点
                // 最后idx存储的是这条索引链表的最高层节点
                for (int i = 1; i <= level; ++i)
                    idx = new Index<K,V>(z, idx, null);
            }
            else { // try to grow by one level
                // 如果新的层数超过了现有跳表的高度
                // 则最多只增加一层
                // 比如现在只有一层索引，那下一次最多增加到两层索引，增加多了也没有意义
                level = max + 1; // hold in array and later pick the one to use
                // idxs用于存储目标节点建立的竖起索引的所有索引节点
                // 其实这里直接使用idx这个最高节点也是可以完成的
                // 只是用一个数组存储所有节点要方便一些
                // 注意，这里数组0号位是没有使用的
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                // 从第一层开始建立一条竖的索引链表
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                //自旋
                for (;;) {
                    //旧的最高层头索引节点
                    h = head;
                    //旧的最高层级
                    int oldLevel = h.level;
                    // 再次检查，如果旧的最高层级已经不比新层级矮了
                    // 说明有其它线程先一步修改了值，从头来过
                    if (level <= oldLevel) // lost race to add level
                        break;
                    // 新的最高层头索引节点
                    HeadIndex<K,V> newh = h;
                    // 头节点指向的数据节点
                    Node<K,V> oldbase = h.node;
                    // 超出的部分建立新的头索引节点
                    for (int j = oldLevel+1; j <= level; ++j)
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    //CAS更新头索引节点
                    if (casHead(h, newh)) {
                        // h指向新的最高层头索引节点
                        h = newh;
                        // 把level赋值为旧的最高层级的
                        // idx指向的不是最高的索引节点了
                        // 而是与旧最高层平齐的索引节点                        
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
            // 经过上面的步骤，有两种情况
            // 一是没有超出高度，新建一条目标节点的索引节点链
            // 二是超出了高度，新建一条目标节点的索引节点链，同时最高层头索引节点同样往上长
            // find insertion points and splice in
            //这时level是等于旧的最高层级的，自旋
            splice: for (int insertionLevel = level;;) {
                int j = h.level;
                for (Index<K,V> q = h, r = q.right, t = idx;;) {
                    if (q == null || t == null)
                        break splice;
                    if (r != null) {
                        Node<K,V> n = r.node;
                        // compare before deletion check avoids needing recheck
                        int c = cpr(cmp, key, n.key);
                        if (n.value == null) {
                            if (!q.unlink(r))
                                break;
                            r = q.right;
                            continue;
                        }
                        if (c > 0) {
                            q = r;
                            r = r.right;
                            continue;
                        }
                    }

                    if (j == insertionLevel) {
                        if (!q.link(r, t))
                            break; // restart
                        if (t.node.value == null) {
                            findNode(key);
                            break splice;
                        }
                        if (--insertionLevel == 0)
                            break splice;
                    }

                    if (--j >= insertionLevel && j < level)
                        t = t.down;
                    q = q.down;
                    r = q.right;
                }
            }
        }
        return null;
    }
    //寻找目标节点之前最近的一个索引对应的数据节点
    private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        //key非空校验
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        //自旋
        for (;;) {
            //向右查找
            for (Index<K,V> q = head, r = q.right, d;;) {
                //右节点不为空
                if (r != null) {
                    Node<K,V> n = r.node;
                    K k = n.key;
                    //右节点的value为空，删除
                    if (n.value == null) {
                        if (!q.unlink(r))
                            //删除失败，跳出
                            break;           // restart
                        //删除后，重新读取右节点
                        r = q.right;         // reread r
                        continue;
                    }
                    //key比右节点的key大，继续向右寻找
                    if (cpr(cmp, key, k) > 0) {
                        //右移
                        q = r;
                        //重新取右节点
                        r = r.right;
                        continue;
                    }
                }
                //如果没有下一级了，就返回这个索引对应的数据节点
                if ((d = q.down) == null)
                    return q.node;
                //下移
                q = d;
                //重新读取右节点
                r = d.right;
            }
        }
    }
    //协助删除
    void helpDelete(Node<K,V> b, Node<K,V> f) {
        /*
         * Rechecking links and then doing only one of the
         * help-out stages per call tends to minimize CAS
         * interference among helping threads.
         */
        //this==n，三者关系是b->n->f
        if (f == next && this == b.next) {
            // 将n的值设置为null后，会先把n的下个节点设置为marker节点
            // 这个marker节点的值是它自己
            // 这里如果不是它自己说明marker失败了，重新marker
            if (f == null || f.value != f) // not already marked
                casNext(f, new Node<K,V>(f));
            else
                // marker过了，就把b的下个节点指向marker的下个节点
                b.casNext(this, f.next);
        }
    }
    //更新当前节点为右节点，当前节点断开
    final boolean unlink(Index<K,V> succ) {
        return node.value != null && casRight(succ, succ.right);
    }    
```
### 添加图解

![image](https://s2.ax1x.com/2019/12/21/QvqGBd.md.png)
假如要插入一个元素9：
1. 寻找目标节点之前最近的一个索引对应的数据节点，在这里也就是找到了5这个数据节点；
2. 从5开始向后遍历，找到目标节点的位置，也就是在8和12之间；
3. 插入9这个元素，Part I 结束；

![image](https://s2.ax1x.com/2019/12/21/QvqruQ.png)
计算其索引层级，假如是3，也就是level=3。
1. 建立竖直的down索引链表；
2. 超过了现有高度2，还要再增加head索引链的高度；
3. 至此，Part II 结束；

![image](https://s2.ax1x.com/2019/12/21/QvqHER.png)

把right指针补齐：
1. 从第3层的head往右找当前层级目标索引的位置；
2. 找到就把目标索引和它前面索引的right指针连上，这里前一个正好是head；
3. 然后前一个索引向下移，这里就是head下移；
4. 再往右找目标索引的位置；
5. 找到了就把right指针连上，这里前一个是3的索引；
6. 然后3的索引下移；
7. 再往右找目标索引的位置；
8. 找到了就把right指针连上，这里前一个是5的索引；
9. 然后5下移，到底了，Part III 结束，整个插入过程结束；

![image](https://s2.ax1x.com/2019/12/21/QvqOC6.png)

### 删除
```java
    public V remove(Object key) {
        return doRemove(key, null);
    }
    final V doRemove(Object key, Object value) {
        //校验key非空
        if (key == null)
            throw new NullPointerException();
        //比较器
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            //寻找目标节点之前的最近的索引节点对应的数据节点
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                //整个链表都遍历完了也没找到目标节点，退出外层循环
                if (n == null)
                    break outer;
                //下一节点
                Node<K,V> f = n.next;
                //检查，说明其它线程修改
                if (n != b.next)                    // inconsistent read
                    //跳出
                    break;
                //value为空，帮助删除
                if ((v = n.value) == null) {        // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                //如果b的值为空或者v等于n，说明b已被删除
                //这时候n就是marker节点，那b就是被删除的那个
                if (b.value == null || v == n)      // b is deleted
                    break;
                //小于，跳出
                if ((c = cpr(cmp, key, n.key)) < 0)
                    break outer;
                //大于，向右寻找
                if (c > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                //c=0，说明n就是要找的元素
                // 如果value不为空且不等于找到元素的value，不需要删除，退出外层循环
                if (value != null && !value.equals(v))
                    break outer;
                //CAS跟新失败，跳出
                if (!n.casValue(v, null))
                    break;
                // 一是如果标记market成功，再尝试将b的下个节点指向下下个节点，如果第二步失败了，进入条件，如果成功了就不用进入条件了
                // 二是如果标记market失败了，直接进入条件
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    //通过findNode()重试删除（里面有个helpDelete()方法）
                    findNode(key);                  // retry via findNode
                else {
                    //说明节点已经删除了，通过findPredecessor()方法删除索引节点
                    findPredecessor(key, cmp);      // clean index
                    if (head.right == null)
                        //如果最高层头索引节点没有右节点，则跳表的高度降级
                        tryReduceLevel();
                }
                //返回删除的元素值
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
        return null;
    }
```
1. 寻找目标节点之前最近的一个索引对应的数据节点（数据节点都是在最底层的链表上）；
2. 从这个数据节点开始往后遍历，直到找到目标节点的位置；
3. 如果这个位置没有元素，直接返回null，表示没有要删除的元素；
4. 如果这个位置有元素，先通过n.casValue(v, null)原子更新把其value设置为null；
5. 通过n.appendMarker(f)在当前元素后面添加一个marker元素标记当前元素是要删除的元素；
6. 通过b.casNext(n, f)尝试删除元素；
7. 如果上面两步中的任意一步失败了都通过findNode(key)中的n.helpDelete(b, f)再去不断尝试删除；
8. 如果上面两步都成功了，再通过findPredecessor(key,cmp)中的q.unlink(r)删除索引节点；
9. 如果head的right指针指向了null，则跳表高度降级；

### 删除图解
删除9这个元素.

![image](https://s2.ax1x.com/2019/12/21/QvOJYt.png)

1. 找到9这个数据节点；
2. 把9这个节点的value值设置为null；
3. 在9后面添加一个marker节点，标记9已经删除了；
4. 让8指向12；
5. 把索引节点与它前一个索引的right断开联系；
6. 跳表高度降级；

为什么要有（2）（3）（4）这么多步骤呢，因为多线程下如果直接让8指向12，可以其它线程先一步在9和12间插入了一个元素10呢，这时候就不对了。

- 如果第（2）步失败了，则直接重试；
- 如果第（3）或（4）步失败了，因为第（2）步是成功的，则通过helpDelete()不断重试去删除；
- 其实helpDelete()里面也是不断地重试（3）和（4）；

只有这三步都正确完成了，才能说明这个元素彻底被删除了。

### 查找

```java
    public V get(Object key) {
        return doGet(key);
    }
    private V doGet(Object key) {
        //key非空校验
        if (key == null)
            throw new NullPointerException();
        //比较器
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            //寻找目标节点之前最近的索引对应的数据节点
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                //如果链表到头还没找到元素，则跳出外层循环
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                //再次校验，不一致读，跳出
                if (n != b.next)                // inconsistent read
                    break;
                //value为空，帮助删除，跳出
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                //如果b的值为空或者v等于n，说明b已被删除
                if (b.value == null || v == n)  // b is deleted
                    break;
                //比较相等，找到元素，返回value
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }
                //如果c<0，说明没找到元素,跳出
                if (c < 0)
                    break outer;
                //c>0，继续找
                //当前节点后移
                b = n;
                //下一个节点后移
                n = f;
            }
        }
        return null;
    }    
```
1. 在底层链表上寻找目标节点之前最近的一个索引对应的数据节点；
2. 从这个数据节点开始往后遍历，直到找到目标节点的位置；
3. 如果这个位置没有元素，直接返回null，表示没有找到元素；
4. 如果这个位置有元素，返回元素的value值；

### 查找图解
要查找9这个元素

![image](https://s2.ax1x.com/2019/12/21/QvXoDg.png)

1. 寻找目标节点之前最近的一个索引对应的数据节点，这里就是5；
2. 从5开始往后遍历，经过8，到9；
3. 找到了返回；

![image](https://s2.ax1x.com/2019/12/21/QvXOCq.md.png)

