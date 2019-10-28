---
layout:     post
title:      "Redo log和Binlog"
date:       2019-10-28 00:00:00
author:     "jiefang"
header-style: text
tags:
    - MySQL
---
## Redo log
Redo log 称为重做日志，用于记录事务操作变化，记录的是数据被修改之后的值。
Redo log 由两部分组成：
- 内存中的重做日志缓冲（redo log buffer）
- 重做日志文件（redo log file）

每次数据更新会先更新 redo log buffer，然后根据 **innodb_flush_log_at_trx_commit** 来控制
redo log buffer 更新到 redo log file 的时机。innodb_flush_log_at_trx_commit 有三个值可
选：
- 0：事务提交时，在事务提交时，每秒触发一次 redo log buffer 写磁盘操作，并调用操作系统fsync 刷新 IO 缓存。
- 1：事务提交时，InnoDB 立即将缓存中的 redo 日志写到日志文件中，并调用操作系统 fsync刷新 IO 缓存；
- 2：事务提交时，InnoDB 立即将缓存中的redo日志写到日志文件中，但不是马上调用 fsync刷新 IO 缓存，而是每秒只做一次磁盘 IO 缓存刷新操作。
- 
innodb_flush_log_at_trx_commit参数的默认值是1，也就是每个事务提交的时候都会从 log buffer 写更新记录到日志文件，而且会刷新磁盘缓存，这完全满足事务持久化的要求，是最安全的，但是这样会有比较大的性能损失。

## Binlog
二进制日志（binlog）记录了所有的 DDL（数据定义语句）和 DML（数据操纵语句），但是不包括 select 和 show 这类操作。Binlog 有以下几个作用：
- 恢复：数据恢复时可以使用二进制日志
- 复制：通过传输二进制日志到从库，然后进行恢复，以实现主从同步
- 审计：可以通过二进制日志进行审计数据的变更操作

可以通过参数 **sync_binlog** 来控制累积多少个事务后才将二进制日志 fsync 到磁盘。
- sync_binlog=0，表示每次提交事务都只write，不fsync
- sync_binlog=1，表示每次提交事务都会执行fsync
- sync_binlog=N(N>1)，表示每次提交事务都write，累积N个事务后才fsync

比如要加快写入数据的速度或者机器磁盘 IO 瓶颈时，可以将 sync_binlog 设置成大于 1 的值，但是如果设置为 N(N>1)时，如果数据库崩溃，可能会丢失最近 N 个事务的 binlog。
