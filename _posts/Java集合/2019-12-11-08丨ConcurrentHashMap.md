---
layout:     post
title:      "Java集合-08丨ConcurrentHashMap"
date:       2019-12-11 16:12:59
author:     "jiefang"
header-style: text
tags:
    - Java集合
    - JUC
---
# ConcurrentHashMap
## 问题
1. ConcurrentHashMap的数据结构？
2. ConcurrentHashMap是怎么解决并发安全问题？
3. ConcurrentHashMap使用了哪些锁？
4. ConcurrentHashMap的扩容？
5. ConcurrentHashMap是否是强一致性的？

## 简介
>ConcurrentHashMap是HashMap的线程安全版本，内部也是使用（数组 + 链表 + 红黑树）的结构来存储元素。相比于同样线程安全的HashTable来说，效率等各方面都有极大地提高。

![image](https://s2.ax1x.com/2019/12/11/QrsfbD.png)

ConcurrentHashMap、HashMap和HashTable的区别：
- HashMap 是非线程安全的哈希表，常用于单线程程序中。
- Hashtable 是线程安全的哈希表，由于是通过内置锁 synchronized 来保证线程安全，在资源争用比较高的环境下，Hashtable 的效率比较低。
- ConcurrentHashMap是一个支持并发操作的线程安全的HashMap，但是他不允许存储空key或value。使用CAS+synchronized来保证并发安全，在并发访问时不需要阻塞线程，所以效率是比Hashtable 要高的。


## 源码

### 属性
ConcurrentHashMap的属性基本与HashMap的相同，多出了一些属性。
```
    //最大容量为2的30次方
    private static final int MAXIMUM_CAPACITY = 1 << 30;
    //默认初始容量16-必须为2的幂
    private static final int DEFAULT_CAPACITY = 16;
    //数组最大容量
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //默认并发度，兼容1.7及之前版本
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    //默认负载因子
    private static final float LOAD_FACTOR = 0.75f;
    //当一个桶中的元素个数大于等于8时链表转化为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    //当一个桶中的元素个数小于等于6时红黑树转化为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    //当数组容量大于 64 时，链表才会转化成红黑树
    static final int MIN_TREEIFY_CAPACITY = 64;
    //扩容时任务的最小转移节点数
    private static final int MIN_TRANSFER_STRIDE = 16;
    //sizeCtl中记录stamp的位数
    private static int RESIZE_STAMP_BITS = 16;
    //帮助扩容的最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    //32-16=16，sizeCtl中记录size大小的偏移量
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    //forwarding nodes的hash值
    static final int MOVED     = -1; // hash for forwarding nodes
    //树根节点的hash值
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
    //CPU数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    //存放Node的数组
    transient volatile Node<K,V>[] table;
    //过渡使用的数组，扩容时使用
    private transient volatile Node<K,V>[] nextTable;
    //基础计数器值(size = baseCount + CounterCell[i].value)
    private transient volatile long baseCount;
    //控制table初始化和扩容操作
    private transient volatile int sizeCtl;
    //节点转移时下一个需要转移的table索引
    private transient volatile int transferIndex;
    //元素变化时用于控制自旋
    private transient volatile int cellsBusy;
    //保存table中的每个节点的元素个数 2的幂次方
    private transient volatile CounterCell[] counterCells;
```
几个重要概念：
- **table**：用来存放Node节点数据的，默认为null，默认大小为16的数组，每次扩容时大小总是2的幂次方；
- **nextTable**：扩容时的新数组为table的两倍；
- **Node**：节点，保存key-value的数据结构；
- **ForwardingNode**：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点已经被移动；
- **ReservationNode**：在`computeIfAbsent`和`compute`方法计算时当做一个占位节点，表示当前节点已经被占用，在`compute`或`computeIfAbsent`的 function 计算完成后插入元素。hash值为-3。
- **sizeCtl**：控制标识符，用来控制table初始化和扩容操作的，代替，含义如下：
    - 负数代表正在进行初始化或扩容操作
    - -1代表正在初始化
    - -N 表示有N-1个线程正在进行扩容操作
    - 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小


### 新增
####  put(K key, V value)

```
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
```
#### putVal(K key, V value, boolean onlyIfAbsent)

```
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //ConcurrentHashMap的key和value都不能为null
        if (key == null || value == null) throw new NullPointerException();
        //计算hash值
        int hash = spread(key.hashCode());
        //要插入的元素所在桶的元素个数
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                //初始化数组
                tab = initTable();
            //数组索引处为空，不加锁直接CAS添加
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果要插入的元素所在的桶的第一个元素的hash是MOVED，则当前线程帮忙一起迁移元素
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                //对数组下标所在元素加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    //key相等，替换旧值
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //新节点添加在链尾
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }//如果是树节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            //新增或查找树节点
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                //判断是否需要转化为树或者扩容
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
#### initTable()
```
    //初始化数组
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            //sizeCtl<0说明正在扩容或初始化，则释放CPU运行即可
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            //CAS修改sizeCtl为-1，初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        //0.75*n 设置扩容阈值n-n/4
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //初始化sizeCtl=0.75*n
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
#### helpTransfer(Node<K,V>[] tab, Node<K,V> f)
```
    //帮助转移节点
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        //如果表非空 且 当前节点是ForwardingNode节点 且 nextTable不为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                //CAS更新帮助转移的线程数
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```
#### treeifyBin(Node<K,V>[] tab, int index)
```
    //扩容或转化为树节点
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```
#### tryPresize(int size)
```
    //尝试扩容
    private final void tryPresize(int size) {
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            //初始化数组操作
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            else if (tab == table) {
                int rs = resizeStamp(n);
                //正在进行扩容操作
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```
#### transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)
移动和/或复制每个节点到新表
```
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //数组长度/8/CPU数（最小为16）
        //目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU（一个线程）处理 16 个
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                //长度为当前长度2倍的新数组
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            // 更新转移下标，就是 老数组的 length
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //创建一个 ForwardingNode 节点，用于占位。
        //当别的线程发现这个槽位中是 ForwardingNode 类型的节点，则跳过这个节点。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），
        // 反之，如果是 false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
        boolean advance = true;
        //是否完成
        boolean finishing = false; // to ensure sweep before committing nextTab
        //i 表示数组下标，bound 表示当前线程可以处理的当前桶区间最小下标
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 如果当前线程可以向后推进；这个循环就是控制 i 递减。
            // 同时，每个线程都会进入这里取得自己需要转移的桶的区间
            while (advance) {
                int nextIndex, nextBound;
                //对 i 减一，判断是否大于等于 bound
                if (--i >= bound || finishing)
                    advance = false;
                // 如果小于等于0，说明没有区间了 ，i 改成 -1，推进状态变成 false，
                // 不再推进，表示，扩容结束了，当前线程可以退出了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }//CAS修改transferIndex的值，每次减16直到0
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //完成扩容
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    //更新扩容阈值
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }//尝试将 sc -1. 表示这个线程结束帮助扩容了，将 sc 的低 16 位减1。
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 如果 sc - 2 不等于标识符左移 16 位。
                    //如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    //扩容结束
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }//获取老数组i下标位置的元素，如果是 null，就使用 fwd 占位。
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 说明别的线程已经处理过了，再次推进一个下标
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                //对数组下标元素加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        //链表方式处理
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            //取模，分为高低两个链表
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //设置低位链表在新数组i下标处
                            setTabAt(nextTab, i, ln);
                            //设置高位链表在新数组i+n下标处
                            setTabAt(nextTab, i + n, hn);
                            // 将旧的链表处设置成占位符
                            setTabAt(tab, i, fwd);
                            //继续向后推进
                            advance = true;
                        }//红黑树方式处理
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            //如果树节点小于等于6转化为链表
                            //低位树
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            //高位树
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            //设置低位树在新数组下标i处
                            setTabAt(nextTab, i, ln);
                            //设置高位树在新数组下标i+n处
                            setTabAt(nextTab, i + n, hn);
                            //旧数组下标i处设置占位
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```
#### transfer()图解
![image](https://s2.ax1x.com/2019/12/11/QsAsnH.png)

- ConcurrentHashMap支持并发扩容，实现方式是，将表拆分，让每个线程处理自己的区间

![image](https://s2.ax1x.com/2019/12/11/QsAgAI.png)

![image](https://s2.ax1x.com/2019/12/11/QsATBj.md.png)

![image](https://s2.ax1x.com/2019/12/11/QsAL40.md.png)


## 总结
