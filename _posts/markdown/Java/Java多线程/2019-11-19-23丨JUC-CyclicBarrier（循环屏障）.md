---
layout:     post
title:      "Java多线程-23丨JUC-CyclicBarrier（循环屏障）"
date:       2019-11-19 14:30:54
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# CyclicBarrier

## 简介

官方定义：
>A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.  CyclicBarriers are useful in programs involving a fixed sized party of threads that must occasionally wait for each other. The barrier is called <em>cyclic</em> because it can be re-used after the waiting threads are released.

翻译后，如下：
- `CyclicBarrier`是一个同步辅助类，它允许一组线程相互等待直到所有线程都到达一个公共的屏障点。
- 在程序中有固定数量的线程，这些线程有时候必须等待彼此，这种情况下，使用`CyclicBarrier`很有帮助。
- 这个**屏障**之所以用**循环**修饰，是因为在所有的线程释放彼此之后，这个屏障是可以重新使用的。

## 原理

![image](https://s2.ax1x.com/2019/11/18/MckRSA.png)
## 源码

```java
    //内部类
    private static class Generation {
        boolean broken = false;
    }
    //守护屏障的锁
    private final ReentrantLock lock = new ReentrantLock();
    //等待条件，直到所有线程到达barrier
    private final Condition trip = lock.newCondition();
    //要屏障的线程数
    private final int parties;
    //当线程都到达barrier，运行的 barrierCommand
    private final Runnable barrierCommand;
    //当前的时代
    private Generation generation = new Generation();
    //还需要等待的线程数，初始值为parties，每当一个线程到来就减一，如果该值为0，则说明所有的线程都到齐了
    private int count;
    //构造函数，制定参与线程数
    public CyclicBarrier(int parties) {
        this(parties, null);
    }
    //构造函数，所有线程到达barrier之后执行给定的barrierAction逻辑
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
    //等待所有参与者线程都到达屏障
    public int await();
    //等待所有的参与者到达barrier，或等待给定的时间
    public int await(long timeout, TimeUnit unit);
    //获取参与等待到达barrier的线程数
    public int getParties() {
        return parties;
    }
    //查询barrier是否处于broken状态
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }
    //打破现有的屏障，让所有线程通过
    private void breakBarrier() {
        //标记broken状态
        generation.broken = true;
        //count恢复初始值
        count = parties;
        // 唤醒当前这一代中所有等待在条件队列里的线程
        trip.signalAll();
    }
    //重置此屏障
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //打破屏障
            breakBarrier();   // break the current generation
            //开启下一代
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
    //开启下一代
    private void nextGeneration() {
        //唤醒此信号上所有wait线程
        trip.signalAll();
        //设置下一代
        count = parties;
        generation = new Generation();
    }
    //返回当前等待barrier的线程数量
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
```
#### await()
```java
    //不带超时
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
    //带超时
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }
    //核心方法
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        //调用await方法需要先得到锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;
            //调用breakBarrier会将当前“代”的broken属性设为true
            //如果一个正在await的线程发现barrier已经被break了，则将直接抛出BrokenBarrierException异常
            if (g.broken)
                throw new BrokenBarrierException();
            //如果当前线程被中断了，则先将屏障打破，再抛出InterruptedException
            // 这么做的原因是，所以等待在barrier的线程都是相互等待的，如果其中一个被中断了，那其他的就不用等了
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }
            //当前线程已经来到了屏障前，先将等待的线程数减一
            int index = --count;
            //如果等待的线程数为0了，说明所有的parties都到齐了
            // 则可以唤醒所有等待的线程，一起通过屏障，并重置屏障
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    //如果传入的barrierCommand不为null，执行它
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //唤醒所有线程，开启新一代
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            //如果count数不为0，就将当前线程挂起，直到所有的线程到齐，或者超时，或者中断发生
            for (;;) {
                try {
                    //没有设定超时，直接在此condition上await
                    if (!timed)
                        trip.await();
                    // 如果设了超时，则等待指定的时间
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    //线程中断了，如果线程被中断时还处于当前这一“代”，并且当前这一代还没有被broken,则先打破屏障
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                    // 一种是g!=generation，说明新的一代已经产生了，没有必要处理这个中断，只要再自我中断一下就好，交给后续的人处理
                    // 一种是g.broken = true, 说明中断前屏障已经被打破了，既然中断发生时屏障已经被打破了，也没有必要再处理这个中断
                        Thread.currentThread().interrupt();
                    }
                }
            
                if (g.broken)
                    throw new BrokenBarrierException();
                // 如果线程被唤醒时，新一代已经被开启了，说明一切正常，直接返回
                if (g != generation)
                    return index;
                // 如果是因为超时时间到了被唤醒，则打破屏障，返回TimeoutException
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```
## 示例
```java
public class CyclicBarrierTest {
    public static void main(String[] args) throws  Exception{
        CyclicBarrier barrier = new CyclicBarrier(5, () -> {
            System.out.println("----线程执行完毕了---");
        });
        for(int i=0;i<5;i++){
            new CyclicBarrierThread(barrier).start();
        }
    }
    static class CyclicBarrierThread extends Thread{
        private CyclicBarrier barrier;

        public CyclicBarrierThread(CyclicBarrier barrier) {
            this.barrier = barrier;
        }

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + "到了");
            try {
                barrier.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```
![执行结果](https://s2.ax1x.com/2019/11/18/Mc8MlV.png)
## 总结
- `CyclicBarrier`实现了类似`CountDownLatch`的逻辑，它可以使得一组线程之间相互等待，直到所有的线程都到齐了之后再继续往下执行。
- `CyclicBarrier`基于**条件队列和独占锁**来实现，**而非共享锁**。
- `CyclicBarrier`可重复使用，在所有线程都到齐了一起通过后，将会开启新的一代。
- `CyclicBarrier`所有互相等待的线程，要么一起通过barrier，要么一个都不要通过，如果有一个线程因为中断，失败或者超时而过早的离开了barrier，则该barrier会被broken掉，所有等待在该barrier上的线程都会抛出BrokenBarrierException（或者InterruptedException）。