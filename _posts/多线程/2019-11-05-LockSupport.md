---
layout:     post
title:      "LockSupport"
date:       2019-11-05 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
---
# LockSupport
>LockSupport是用来创建锁和其他同步类的基本线程阻塞原语

## 使用
每个使用LockSupport的线程都会与一个许可关联，如果该许可可用，并且可在进程中使用，则调用park()将会立即返回，否则可能阻塞。如果许可尚不可用，则可以调用 unpark 使其可用。但是注意许可不可重入，也就是说只能调用一次park()方法，否则会一直阻塞。


方法名 | 描述
---|---
 park()| 阻塞当前线程，如果掉用unpark(Thread)方法或被中断，才能从park()返回
 void parkNanos(long nanos)| 阻塞当前线程，超时返回，阻塞时间最长不超过nanos纳秒
 void parkUntil(long deadline)|阻塞当前线程，直到deadline时间点
 void unpark(Thread)|唤醒处于阻塞状态的线程
 
源码：
```
public static void park() {
    UNSAFE.park(false, 0L);
}

public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```
### 特色
- 可以先unpark再park：相当于unpark时提供了一个许可，park时直接消耗这个许可，不会真正的阻塞。

## 原理
每个线程都有Parker实例,如下面的代码所示：
```
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
LockSupport就是通过控制变量`_counter`来对线程阻塞唤醒进行控制的。原理有点类似于信号量机制。
- 当调用`park()`方法时，会将_counter置为0，同时判断前值，小于1说明前面被`unpark`过,则直接退出，否则将使该线程阻塞。
- 当调用`unpark()`方法时，会将_counter置为1，同时判断前值，小于1会进行线程唤醒，否则直接退出。
形象的理解，线程阻塞需要消耗凭证(permit)，这个凭证最多只有1个。当调用park方法时，如果有凭证，则会直接消耗掉这个凭证然后正常退出；但是如果没有凭证，就必须阻塞等待凭证可用；而unpark则相反，它会增加一个凭证，但凭证最多只能有1个。
- 为什么可以先唤醒线程后阻塞线程？</br>
因为unpark获得了一个凭证,之后调用park因为有凭证消费，故不会阻塞。
- 为什么唤醒两次后阻塞两次会阻塞线程。</br>
因为凭证的数量最多为1，连续调用两次unpark和调用一次unpark效果一样，只会增加一个凭证；而调用两次park却需要消费两个凭证。

## 总结
LockSupport是JDK中用来实现线程阻塞和唤醒的工具。使用它可以在任何场合使线程阻塞，可以指定任何线程进行唤醒，并且不用担心阻塞和唤醒操作的顺序，但要注意连续多次唤醒的效果和一次唤醒是一样的。