---
layout:     post
title:      "01丨深入分析synchronized的实现原理"
date:       2019-10-28 00:00:00
author:     "jiefang"
header-style: text
tags:
    - 多线程
    - 锁
---
# 深入分析synchronized的实现原理

* [深入分析synchronized的实现原理](#深入分析synchronized的实现原理)
	* [实现原理](#实现原理)
	* [Java对象头](#java对象头)
	* [Monitor](#monitor)
		* [ObjectMonitor](#objectmonitor)

## 实现原理
**synchronized**
可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性。
Java 中每一个对象都可以作为锁，这是 synchronized 实现同步的基础：

- 普通同步方法，锁是当前实例对象
- 静态同步方法，锁是当前类的 class 对象
- 同步方法块，锁是括号里面的对象

- 1）同步代码块是使用 monitorenter 和 monitorexit 指令实现的；
- 2）同步方法依靠的是方法修饰符上的ACC_SYNCHRONIZED 实现。
- 
**同步代码块**：monitorenter 指令插入到同步代码块的开始位置，monitorexit 指令插入到同步代码块的结束位置，JVM 需要保证每一个 monitorenter 都有一个 monitorexit 与之相对应。任何对象都有一个 Monitor 与之相关联，当且一个 Monitor 被持有之后，他将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 Monitor 所有权，即尝试获取对象的锁。

## Java对象头
对象由多部分构成的，对象头，属性字段、补齐区域等。所谓补齐区域是指如果对象总大小不是4字节的整数倍，会填充上一段内存地址使之成为整数倍。
对象头这部分在对象的最前端，包含两部分或者三部分：Mark Word（标记字段）和 Klass Pointer（类型指针） ，如果对象是一个数组，那么还可能包含第三部分：数组的长度。

![OOP-Klass模型](https://s2.ax1x.com/2019/10/18/KZJn39.md.png)

Klass Pointer 是是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
Mark Word 用于存储对象自身的运行时数据，主要包含对象的哈希值、年龄分代、锁标志位等，大小为32位或64位，当对象处于不同的锁状态时，它的Mark Word里的东西会被替换成不同东西，如下表所示：
![32位Mark Word](https://s2.ax1x.com/2019/10/24/KNjzMq.md.png)

 ![64位Mark Word](https://s2.ax1x.com/2019/10/18/KZJhD0.md.png)
 
 ## Monitor
 什么是Monitor？
- 1.Monitor是一种用来实现同步的工具
- 2.与每个java对象相关联，即每个java对象都有一个Monitor与之对应
- 3.Monitor是实现Sychronized(内置锁)的基础

### ObjectMonitor
在HotSpot虚拟机中，monitor采用ObjectMonitor实现。

![ObjectMonitor](https://s2.ax1x.com/2019/10/25/KaxF76.png)

ObjectMonitor对象中有两个队列：**_WaitSet** 和 **_EntryList**，用来保存ObjectWaiter对象列表；**_owner**指向获得ObjectMonitor对象的线程。

![Monitor Object 设计模式](https://s2.ax1x.com/2019/10/25/KaxggJ.png)

_WaitSet：处于wait状态的线程，会被加入到wait set；
_EntryList：处于等待锁block状态的线程，会被加入到entry set；


![脑图](https://s2.ax1x.com/2019/10/25/Kd0xcq.png)

[jdk源码剖析二: 对象内存布局、synchronized终极原理](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label4_1)

[Java Monitor（管程）](https://segmentfault.com/a/1190000019494432)

[JAVA并发-Monitor简介](https://blog.csdn.net/ignorewho/article/details/80854625)

[关于synchronized的Monitor Object机制的研究](https://blog.csdn.net/m_xiaoer/article/details/73274642)


[程序员应该死磕的Monitor知识点](http://baijiahao.baidu.com/s?id=1639857097437674576&wfr=spider&for=pc)

[Java对象头与锁](https://www.cnblogs.com/ZoHy/p/11313155.html)

[冷静分析 Synchronized（上）](https://juejin.im/post/5abc9e14f265da23953111d6#heading-16)

[冷静分析 Synchronized（下）](https://juejin.im/post/5abc9de851882555770c8c72#heading-14)

[深入JVM锁机制-synchronized](https://blog.csdn.net/chen77716/article/details/6618779)