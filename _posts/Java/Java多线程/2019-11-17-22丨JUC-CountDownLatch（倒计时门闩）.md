---
layout:     post
title:      "Java多线程-22丨JUC-CountDownLatch（倒计时门闩）"
date:       2019-11-17 22:04:24
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# JUC-CountDownLatch

## 简介
>`CountDownLatch`是一个同步辅助类，通过AQS实现的一个闭锁。在其他线程完成它们的操作之前，允许一个多个线程等待。简单来说，`CountDownLatch`中有一个锁计数，在计数到达0之前，线程会一直等待。

![原理图](https://s2.ax1x.com/2019/11/17/MrLih4.md.png)

## 用法
```java
    //初始化
    CountDownLatch countDownLatch = new CountDownLatch(10);
    for (int i = 0; i < 10; i++) {
        Executors.newFixedThreadPool(10).execute(()->{
            System.out.println("");
            //计数减1
            countDownLatch.countDown();
        });
    }
    //主线程阻塞
    countDownLatch.await();
```
## 原理

### 类图
`CountDownLatch`是一个“共享锁”，内部定义了自己的同步器`Sync`，`Sync`继承自AQS，实现了`tryAcquireShared`和`tryReleaseShared`两个方法。需要注意的是，`CountDownLatch`中的锁是响应中断的，如果线程在对锁进行操作期间发生中断，会直接抛出`InterruptedException`

![image](https://s2.ax1x.com/2019/11/17/MsSuM4.png)
### 源码
#### `CountDownLatch.Sync`源码

```java
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;
        //构造函数，设置AQS的state
        Sync(int count) {
            setState(count);
        }
        //获取AQS的state
        int getCount() {
            return getState();
        }
        //AQS中tryAcquireShared的子类实现
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
        //AQS中tryReleaseShared的子类实现
        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                //该方法只有在count值原来不为0，但是调用后变为0时，才会返回true，否则返回false
                if (c == 0)
                    return false;
                int nextc = c-1;
                //state值从大于0变为0值时会返回true
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```

#### CountDownLatch构造函数
```java
    //CountDownLatch构造函数
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
    //Sync构造函数，设置AQS state
    Sync(int count) {
        setState(count);
    }
```
#### await()
```java
    //使当前线程等待，直到锁存器计数到零为止，除非该线程{@linkplain Thread＃interrupt interrupted}。
    public void await() throws InterruptedException {
        //
        sync.acquireSharedInterruptibly(1);
    }
    //AQS中获取共享锁响应中断
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        //交由子类实现
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

#### CountDownLatch()
```java
    //减少锁存器的计数，如果计数达到零，则释放所有等待线程
    public void countDown() {
        sync.releaseShared(1);
    }
    //AQS释放共享锁
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }    
```
## 总结

- **`CountDownLatch`内部通过共享锁实现**。
- 在创建`CountDownLatch`实例时，需要传递一个int型的参数：count，该参数为计数器的初始值，也可以理解为该共享锁可以获取的总次数。
- 当某个线程调用`await()`方法，程序首先判断count的值是否为0，如果不会0的话则会一直等待直到为0为止。当其他线程调用countDown()方法时，则执行释放共享锁状态，使count - 1。
- 当在创建`CountDownLatch`时初始化的count参数，必须要有count线程调用`countDown()`方法才会使计数器count等于0，锁才会释放，前面等待的线程才会继续运行。
- **`CountDownLatch`不能重置**。