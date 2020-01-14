---
layout:     post
title:      "Redis-12丨Redis的RDB持久化"
date:       2020-01-14 22:40:29
author:     "jiefang"
header-style: text
tags:
    - Redis
---
# Redis的RDB持久化
RDB持久化是将当前进程中的数据生成快照保存到硬盘(因此也称作快照持久化)，保存的文件后缀是rdb；当Redis重新启动时，可以读取快照文件恢复数据。
## 手动触发
`save`命令和`bgsave`命令都可以生成RDB文件。
`save`命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在Redis服务器阻塞期间，服务器不能处理任何命令请求。
`bgsave`命令会创建一个子进程，由子进程来负责创建RDB文件，父进程(即Redis主进程)则继续处理请求。

- `bgsave`命令执行过程中，只有`fork`子进程时会阻塞服务器，而对于save命令，整个过程都会阻塞服务器，因此`save`已基本被废弃，线上环境要杜绝save的使用；
- 自动触发RDB持久化时，Redis也会选择bgsave而不是save来进行持久化；

## 自动触发
`save m n`:自动触发最常见的情况是在配置文件中通过`save m n`，指定当m秒内发生n次变化时，会触发`bgsave`。

![image](https://s2.ax1x.com/2020/01/14/lLFa1U.png)

其中`save 900 1`的含义是：当时间到900秒时，如果redis数据发生了至少1次变化，则执行`bgsave`；`save 300 10`和`save 60 10000`同理。当三个save条件满足任意一个时，都会引起bgsave的调用。

## save m n的实现原理

**Redis的save m n，是通过`serverCron`函数、dirty计数器、和`lastsave`时间戳来实现的。**

- `serverCron`是Redis服务器的周期性操作函数，默认每隔100ms执行一次；该函数对服务器的状态进行维护，其中一项工作就是检查 `save m n` 配置的条件是否满足，如果满足就执行`bgsave`。
- `dirty计数器`是Redis服务器维持的一个状态，记录了上一次执行`bgsave/save`命令后，服务器状态进行了多少次修改(包括增删改)；而当`save/bgsave`执行完成后，会将dirty重新置为0。
>例如，如果Redis执行了set mykey helloworld，则dirty值会+1；如果执行了sadd myset v1 v2 v3，则dirty值会+3；注意dirty记录的是服务器进行了多少次修改，而不是客户端执行了多少修改数据的命令。
- `lastsave时间戳`也是Redis服务器维持的一个状态，记录的是上一次成功执行`save/bgsave`的时间。
- **save m n的原理**如下：每隔100ms，执行`serverCron函数`；在serverCron函数中，遍历save m n配置的保存条件，只要有一个条件满足，就进行bgsave。对于每一个save m n条件，只有下面两条同时满足时才算满足：
    - 当前时间-lastsave > m
    - dirty >= n

除了`save m n `以外，还有一些其他情况会触发bgsave：
- 在主从复制场景下，如果从节点执行全量复制操作，则主节点会执行`bgsave`命令，并将rdb文件发送给从节点
- 执行`shutdown命令`时，自动执行rdb持久化
- 执行`debug reload命令`重新加载Redis时，也会自动触发`bgsave`操作。

### 执行流程
![image](https://s2.ax1x.com/2020/01/14/lLAUL4.png)

1. Redis父进程首先判断：当前是否在执行`save`，或`bgsave/bgrewriteaof`的子进程，如果在执行则`bgsave`命令直接返回。`bgsave/bgrewriteaof`的子进程不能同时执行，主要是基于性能方面的考虑：**两个并发的子进程同时执行大量的磁盘写操作，可能引起严重的性能问题**。
2. 父进程执行fork操作创建子进程，这个过程中父进程是阻塞的，Redis不能执行来自客户端的任何命令。
3. 父进程fork后，`bgsave`命令返回”Background saving started”信息并不再阻塞父进程，并可以响应其他命令。
4. 子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换。
5. 子进程发送信号给父进程表示完成，父进程更新统计信息。

## RDB常用配置

- `save m n`：`bgsave`自动触发的条件；如果没有`save m n`配置，相当于自动的RDB持久化关闭，不过此时仍可以通过其他方式触发;
- `stop-writes-on-bgsave-error yes`：当`bgsave`出现错误时，Redis是否停止执行写命令；设置为yes，则当硬盘出现问题时，可以及时发现，避免数据的大量丢失；设置为no，则Redis无视`bgsave`的错误继续执行写命令，当对Redis服务器的系统(尤其是硬盘)使用了监控时，该选项考虑设置为no;
- `rdbcompression yes`：是否开启RDB文件压缩;
- `rdbchecksum yes`：是否开启RDB文件的校验，在写入文件和读取文件时都起作用；关闭checksum在写入文件和启动文件时大约能带来10%的性能提升，但是数据损坏时无法发现;
- `dbfilename dump.rdb`：RDB文件名;
- `dir ./`：RDB文件和AOF文件所在目录;

## RDB优缺点
### 优点

- RDB是一个紧凑压缩的二进制文件，代表Redis在某一个时间点上的数据快照。非常适合用于备份，全量复制等场景。比如每6小时执行bgsave备份，并把RDB文件拷贝到远程机器，用于灾难恢复。
- Redis加载RDB恢复数据远远快于AOF方式。

### 缺点

- RDB方式数据没办法做到实时持久化/秒级持久化。因为bgsave每次运行都要执行fork操作创建子进程，属于重量级操作，频繁执行成本过高。
- RDB文件使用特定二进制格式保存，Redis版本演进过程中有多个格式的RDB版本，老版本Redis服务无法兼容新版RDB格式。