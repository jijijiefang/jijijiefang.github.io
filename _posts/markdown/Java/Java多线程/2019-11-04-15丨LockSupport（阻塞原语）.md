---
layout:     post
title:      "Java多线程-15丨LockSupport（阻塞原语）"
date:       2019-11-05 00:00:00
author:     "jiefang"
header-style: text
tags:
    - Java多线程
    - JUC
---
# LockSupport
>LockSupport是用来创建锁和其他同步类的基本线程阻塞原语

## 使用
每个使用`LockSupport`的线程都会与一个许可关联，如果该许可可用，并且可在进程中使用，则调用`park()`将会立即返回，否则可能阻塞。如果许可尚不可用，则可以调用 `unpark()`使其可用。但是注意许可不可重入，也就是说只能调用一次`park()`方法，否则会一直阻塞。


方法名 | 描述
---|---
 `park()` | 阻塞当前线程，如果调用`unpark(Thread)`方法或被中断，才能从`park()`返回 
 `void parkNanos(long nanos)` | 阻塞当前线程，超时返回，阻塞时间最长不超过nanos纳秒
 `void parkUntil(long deadline)` |阻塞当前线程，直到`deadline`时间点
 `void unpark(Thread)` |唤醒处于阻塞状态的线程

源码：
```java
public static void park() {
    UNSAFE.park(false, 0L);
}

public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```
### 特色
- 可以先`unpark()`再`park()`：相当于`unpark()`时提供了一个许可，`park()`时直接消耗这个许可，不会真正的阻塞。
### 与wait和notify相比

- `wait`和`notify`都是Object中的方法,在调用这两个方法前必须先获得锁对象，只能在同步代码块中调用；`LockSupport`不需要在同步代码块里。
- `unpark()`函数可以先于`park()`调用；`notify`不能先于`wait`调用，否则会抛异常。
## 原理
每个线程都有Parker实例,如下面的代码所示：
```c++
class Parker : public os::PlatformParker {
private:
  volatile int _counter ;
  ...
public:
  void park(bool isAbsolute, jlong time);
  void unpark();
  ...
}
class PlatformParker : public CHeapObj<mtInternal> {
  protected:
    pthread_mutex_t _mutex [1] ;
    pthread_cond_t  _cond  [1] ;
    ...
}
```
`LockSupport`就是通过控制变量`_counter`来对线程阻塞唤醒进行控制的。原理有点类似于**信号量机制**。

- 当调用`park()`方法时，会将`_counter`置为0，同时判断前值，小于1说明前面被`unpark`过,则直接退出，否则将使该线程阻塞。
- 当调用`unpark()`方法时，会将`_counter`置为1，同时判断前值，小于1会进行线程唤醒，否则直接退出。
形象的理解，线程阻塞需要消耗**凭证(permit)**，这个凭证最多只有1个。当调用`park()`方法时，如果有凭证，则会直接消耗掉这个凭证然后正常退出；但是如果没有凭证，就必须阻塞等待凭证可用；而`unpark()`则相反，它会增加一个凭证，但凭证最多只能有1个。
- 为什么可以先唤醒线程后阻塞线程？</br>
因为`unpark()`获得了一个凭证,之后调用`park()`因为有凭证消费，故不会阻塞。
- 为什么唤醒两次后阻塞两次会阻塞线程。</br>
因为凭证的数量最多为1，连续调用两次`unpark()`和调用一次`unpark()`效果一样，只会增加一个凭证；而调用两次`park()`却需要消费两个凭证。

## 比较
### Thread.sleep()和Object.wait()
最大的区别就是`Thread.sleep()`不会释放锁资源，`Object.wait()`会释放锁资源。
- `Thread.sleep()`不会释放占有的锁，`Object.wait()`会释放占有的锁；
- `Thread.sleep()`必须传入时间，`Object.wait()`可传可不传，不传表示一直阻塞下去；
- `Thread.sleep()`到时间了会自动唤醒，然后继续执行；
- `Object.wait()`不带时间的，需要另一个线程使用`Object.notify()`唤醒；
- `Object.wait()`带时间的，假如没有被`notify`，到时间了会自动唤醒，这时又分好两种情况，一是立即获取到了锁，线程自然会继续执行；二是没有立即获取锁，线程进入同步队列等待获取锁；

### Thread.sleep()和LockSupport.park()
- 从功能上来说，`Thread.sleep()`和`LockSupport.park()`方法类似，都是阻塞当前线程的执行，且都**不会释放当前线程占有的锁资源**；
- `Thread.sleep()`没法从外部唤醒，只能自己醒过来；
- `LockSupport.park()`方法可以被另一个线程调用`LockSupport.unpark()`方法唤醒；
- `Thread.sleep()`方法声明上抛出了`InterruptedException`中断异常，所以调用者需要捕获这个异常或者再抛出；
- `LockSupport.park()`方法**不需要捕获中断异常**；
- `Thread.sleep()`本身就是一个native方法；
- `LockSupport.park()`底层是调用的Unsafe的native方法；

### Object.wait()和LockSupport.park()
- `Object.wait()`方法需要在`synchronized`块中执行；
- `LockSupport.park()`可以在任意地方执行；
- `Object.wait()`方法声明抛出了中断异常，调用者需要处理异常；
- `LockSupport.park()`不需要捕获中断异常；
- `Object.wait()`不带超时的，需要另一个线程执行`notify()`来唤醒，但不一定继续执行后续内容，有可能需要获取锁；
- `LockSupport.park()`不带超时的，需要另一个线程执行`unpark()`来唤醒，一定会继续执行后续内容；
- **在`wait()`之前执行了`notify()`会抛出`IllegalMonitorStateException`异常**；
- **在`park()`之前执行了`unpark()`，线程不会被阻塞，直接跳过`park()`，继续执行后续内容**；

![脑图](https://s2.ax1x.com/2019/12/24/lPBIJJ.png)

## 总结
`LockSupport`是JDK中用来实现线程阻塞和唤醒的工具。使用它可以在任何场合使线程阻塞，可以指定任何线程进行唤醒，并且不用担心阻塞和唤醒操作的顺序，但要注意连续多次唤醒的效果和一次唤醒是一样的。