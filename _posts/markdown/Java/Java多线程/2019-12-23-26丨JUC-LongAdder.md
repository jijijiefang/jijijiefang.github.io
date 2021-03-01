---
layout:     post
title:      "Java多线程-26丨JUC-LongAdder"
date:       2019-12-23 15:07:29
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# LongAdder

## 简介
Java注释
>One or more variables that together maintain an initially zero {@code long} sum.  When updates (method {@link #add}) are contended across threads, the set of variables may grow dynamically to reduce contention. Method {@link #sum} (or, equivalently, {@link #longValue}) returns the current total combined across the variables maintaining the sum.

翻译
>一个或多个变量共同维持最初的零。当跨线程竞争更新`add`时，变量集可能会动态增长以减少竞争。方法`sum`或等效地`longValue`,返回变量中保持总和的当前总数。

`LongAdder`在高并发的场景下会比`AtomicLong`具有更好的性能，代价是消耗更多的内存空间。

### 类图

![LongAdder](https://s2.ax1x.com/2019/12/22/QzgJDf.png)

## 原理
`AtomicLong`是多个线程针对单个热点值value进行原子操作。而`LongAdder`是每个线程拥有自己的槽，各个线程一般只对自己槽中的那个值进行CAS操作。
>比如有三个ThreadA、ThreadB、ThreadC，每个线程对value增加10。

对于`AtomicLong`，最终结果的计算始终是下面这个形式：
```java
    value = 10 + 10 + 10 = 30
```
但是对于`LongAdder`来说，内部有一个`base`变量，一个`Cell[]`数组。
`base`变量：非竞态条件下，直接累加到该变量上
`Cell[]`数组：竞态条件下，累加个各个线程自己的槽`Cell[i]`中
最终结果的计算是下面这个形式：
```java
    value = base + ∑Cell[i]
```

## 源码

### Striped64
`Striped64`是在java8中添加用来支持累加器的并发组件，它可以在并发环境下使用来做某种计数，`Striped64`的设计思路是在竞争激烈的时候尽量分散竞争，在实现上，`Striped64`维护了一个`base`和一个`Cell`数组，计数线程会首先试图更新`base`变量，如果成功则退出计数，否则会认为当前竞争是很激烈的，那么就会通过`Cell`数组来分散计数，`Striped64`根据线程来计算哈希，然后将不同的线程分散到不同的`Cell`数组的`index`上，然后这个线程的计数内容就会保存在该`Cell`的位置上面，基于这种设计，最后的总计数需要结合`base`以及散落在`Cell`数组中的计数内容。
```java
abstract class Striped64 extends Number {
    //Cell内部类
    @sun.misc.Contended static final class Cell {
        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }
        //CAS准备工作
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
    //CPU核数
    static final int NCPU = Runtime.getRuntime().availableProcessors();
    //Cell数组，大小为2的幂
    transient volatile Cell[] cells;
    //基础数
    transient volatile long base;
    //锁的个数
    transient volatile int cellsBusy;

    Striped64() {
    }
    //CAS更新base值
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }
    //CAS修改cellsBusy从0到1
    final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }
    //当前线程的探测值
    static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }
    //伪随机前进并记录给定线程的给定探测值
    static final int advanceProbe(int probe) {
        probe ^= probe << 13;   // xorshift
        probe ^= probe >>> 17;
        probe ^= probe << 5;
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
        return probe;
    }
    //long值累加
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        //如果当前线程的hash值等于0，初始化生成一个非0的hash值
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        //如何hash取模后映射的Cell单元不是null，则为true
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //数组不为空且数组元素数量大于0
            if ((as = cells) != null && (n = as.length) > 0) {
                //hash取模后映射Cell单元如果为空
                if ((a = as[(n - 1) & h]) == null) {
                    //初始化新的Cell元素
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        //如果cellsBusy是0且CAS更新cellsBusy为1成功
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                //再次校验
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                //创建Cell后跟新cellsBusy为0
                                cellsBusy = 0;
                            }
                            //创建成功跳出
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }//wasUncontended表示上一次CAS更新单元是否成功
                else if (!wasUncontended)       // CAS already known to fail
                    //设置为true,重新计算线程hash值
                    wasUncontended = true;      // Continue after rehash
                //计算Cell单元a的值，如果传入方法为空，直接相加；否则根据传入方法计算
                //CAS更新计算后新值成功后跳出
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                //数组长度大于CPU核数或数组已被更新，不在扩容
                else if (n >= NCPU || cells != as)
                    //扩容标识设置false
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                //cellsBusy当前为0且CAS更新cellsBusy值为1成功，进行扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        //数组没有被其它线程更新
                        if (cells == as) {      // Expand table unless stale
                            //2倍扩容
                            Cell[] rs = new Cell[n << 1];
                            //旧数组元素挪到新数组中
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    //使用新数组重试
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }//cellsBusy等于0，没有加锁且没有初始化数组，尝试加锁，并初始化数组
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    //初始化数组
                    if (cells == as) {
                        //初始化数组长度为2（必须为2的幂）
                        Cell[] rs = new Cell[2];
                        //Cell初始化，赋值为x
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }//数组正在进行初始化，则直接尝试在base数值上进行累加操作
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
    //double值累加
    final void doubleAccumulate(double x, DoubleBinaryOperator fn,
                                boolean wasUncontended) {
        int h;
        //如果当前线程的hash值等于0，初始化生成一个非0的hash值
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        //如何hash取模后映射的Cell单元不是null，则为true
        boolean collide = false;                // True if last slot nonempty
        //自旋
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            //数组不为空且数组元素大于0
            if ((as = cells) != null && (n = as.length) > 0) {
                //hash取模后的Cell如果为空
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(Double.doubleToRawLongBits(x));
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value,
                               ((fn == null) ?
                                Double.doubleToRawLongBits
                                (Double.longBitsToDouble(v) + x) :
                                Double.doubleToRawLongBits
                                (fn.applyAsDouble
                                 (Double.longBitsToDouble(v), x)))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(Double.doubleToRawLongBits(x));
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base,
                             ((fn == null) ?
                              Double.doubleToRawLongBits
                              (Double.longBitsToDouble(v) + x) :
                              Double.doubleToRawLongBits
                              (fn.applyAsDouble
                               (Double.longBitsToDouble(v), x)))))
                break;                          // Fall back on using base
        }
    }

    //Striped64的CAS准备
    private static final sun.misc.Unsafe UNSAFE;
    private static final long BASE;
    private static final long CELLSBUSY;
    private static final long PROBE;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> sk = Striped64.class;
            BASE = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("base"));
            CELLSBUSY = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("cellsBusy"));
            Class<?> tk = Thread.class;
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}

```
### LongAdder

```java
public class LongAdder extends Striped64 implements Serializable {
    private static final long serialVersionUID = 7249069246863182397L;
    //空构造器
    public LongAdder() {
    }
    //加long
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        //如果Cell数组不为空或CAS跟新base值失败
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            //取cells数组中的任意一个元素a判断是否为空
            //对a进行cas操作并将执行结果赋值标志位uncontended
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }
    //加1
    public void increment() {
        add(1L);
    }
    //减1
    public void decrement() {
        add(-1L);
    }
    //求和
    //此返回值不是绝对精确的，调用方法时有可能有其它线程正在计数累加，在高并发时是近似值，除非全局加锁才能得到精确值。
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        //如果Cell数组不为空，累加
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
    //重置
    public void reset() {
        Cell[] as = cells; Cell a;
        base = 0L;
        //重置数组
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = 0L;
            }
        }
    }
    //求和并重置
    public long sumThenReset() {
        Cell[] as = cells; Cell a;
        long sum = base;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    sum += a.value;
                    a.value = 0L;
                }
            }
        }
        return sum;
    }
    //转成字符串
    public String toString() {
        return Long.toString(sum());
    }
    //求和
    public long longValue() {
        return sum();
    }
    //求和转成int返回
    public int intValue() {
        return (int)sum();
    }
    //求和转成float返回
    public float floatValue() {
        return (float)sum();
    }
    //求和转成double返回
    public double doubleValue() {
        return (double)sum();
    }
    //序列化代理
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 7249069246863182397L;
        //当前值
        private final long value;
        //构造方法
        SerializationProxy(LongAdder a) {
            value = a.sum();
        }
        //反序列化
        private Object readResolve() {
            LongAdder a = new LongAdder();
            a.base = value;
            return a;
        }
    }

    //序列化
    private Object writeReplace() {
        return new SerializationProxy(this);
    }
    //反序列化
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.InvalidObjectException {
        throw new java.io.InvalidObjectException("Proxy required");
    }
}
```

### LongAccumulator
`LongAccumulator`是`LongAdder`的增强版,`LongAdder`只能针对数值的进行加减运算，而`LongAccumulator`提供了自定义的函数操作。通过`LongBinaryOperator`，可以自定义对入参的任意操作，并返回结果（`LongBinaryOperator.applyAsLong(long left, long right)`接收2个long作为参数，并返回1个long）。`LongAccumulator`内部原理和`LongAdder`几乎完全一样，都是利用了父类`Striped64`的`longAccumulate`方法。
```java
public class LongAccumulator extends Striped64 implements Serializable {
    private static final long serialVersionUID = 7249069246863182397L;
    //通过实现LongBinaryOperator，实现自定义计算
    private final LongBinaryOperator function;
    private final long identity;
    //构造方法
    public LongAccumulator(LongBinaryOperator accumulatorFunction,
                           long identity) {
        this.function = accumulatorFunction;
        base = this.identity = identity;
    }
    //计算
    public void accumulate(long x) {
        Cell[] as; long b, v, r; int m; Cell a;
        if ((as = cells) != null ||
            (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended =
                  (r = function.applyAsLong(v = a.value, x)) == v ||
                  a.cas(v, r)))
                longAccumulate(x, function, uncontended);
        }
    }
    //返回计算结果
    public long get() {
        Cell[] as = cells; Cell a;
        long result = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    result = function.applyAsLong(result, a.value);
            }
        }
        return result;
    }
    //重置
    public void reset() {
        Cell[] as = cells; Cell a;
        base = identity;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = identity;
            }
        }
    }
    //返回结果并重置
    public long getThenReset() {
        Cell[] as = cells; Cell a;
        long result = base;
        base = identity;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    long v = a.value;
                    a.value = identity;
                    result = function.applyAsLong(result, v);
                }
            }
        }
        return result;
    }
    //转化成字符串
    public String toString() {
        return Long.toString(get());
    }
    //返回结果long类型
    public long longValue() {
        return get();
    }
    //返回结果int类型
    public int intValue() {
        return (int)get();
    }
    //返回结果float类型
    public float floatValue() {
        return (float)get();
    }
    //返回结果double类型
    public double doubleValue() {
        return (double)get();
    }
    //序列化代理
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 7249069246863182397L;
        private final long value;
        private final LongBinaryOperator function;
        private final long identity;
        //序列化代理构造方法
        SerializationProxy(LongAccumulator a) {
            function = a.function;
            identity = a.identity;
            value = a.get();
        }
        //反序列化
        private Object readResolve() {
            LongAccumulator a = new LongAccumulator(function, identity);
            a.base = value;
            return a;
        }
    }
    //序列化
    private Object writeReplace() {
        return new SerializationProxy(this);
    }
    //反序列化
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.InvalidObjectException {
        throw new java.io.InvalidObjectException("Proxy required");
    }
}
```
## 总结
`LongAdder`类与`AtomicLong`类的区别在于高并发时前者将对单一变量的CAS操作分散为对数组cells中多个元素的CAS操作，取值时进行求和；而在并发较低时仅对base变量进行CAS操作，与`AtomicLong`类原理相同。