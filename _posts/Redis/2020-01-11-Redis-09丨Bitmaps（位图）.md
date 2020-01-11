---
layout:     post
title:      "Redis-09丨Bitmaps（位图）"
date:       2020-01-11 20:38:41
author:     "jiefang"
header-style: text
tags:
    - Redis
---
# Bitmaps（位图）
Bitmaps 并不是实际的数据类型，而是定义在String类型上的一个面向字节操作的集合。`Bitmaps`就是通过最小的单位bit来进行0或者1的设置，表示某个元素对应的值或者状态。
一个bit的值，或者是0，或者是1；也就是说一个bit能存储的最多信息是2。
## 优势和限制
### 优势
- 基于最小的单位bit进行存储，所以非常省空间。
- 设置时候时间复杂度O(1)、读取时候时间复杂度O(n)，操作是非常快的。
- 二进制数据的存储，进行相关计算的时候非常快。
- 方便扩容

### 限制
redis中bit映射被限制在512MB之内，所以最大是2^32位。建议每个key的位数都控制下，因为读取时候时间复杂度O(n)，越大的串读的时间花销越多。

## 基本使用

### getbit key offset
- 获取位图指定索引的值
```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> getbit hello 0
(integer) 0
127.0.0.1:6379> getbit hello 1
(integer) 1
127.0.0.1:6379> getbit hello 2
(integer) 1
```
![image](https://s2.ax1x.com/2020/01/11/lIJgL8.md.png)
### setbit key offset value
- 给位图指定索引设置值，返回该索引位置的原始值
```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> getbit hello 7
(integer) 0
127.0.0.1:6379> setbit hello 7 1
(integer) 0
127.0.0.1:6379> get hello
"cig"

127.0.0.1:6379> setbit world 50 1
(integer) 0
127.0.0.1:6379> get world
"\x00\x00\x00\x00\x00\x00 "
127.0.0.1:6379> setbit world 50 0
(integer) 1
127.0.0.1:6379> get world
"\x00\x00\x00\x00\x00\x00\x00"
```
### bitcount key [start end]
- 获取位图指定范围（start到end，单位为字节，如果不指定就是获取全部）位值为1的个数。
- 范围必须是 8 的倍数。
```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> bitcount hello
(integer) 12
127.0.0.1:6379> setbit hello 7 1
(integer) 0
127.0.0.1:6379> bitcount hello
(integer) 13

127.0.0.1:6379> bitcount hello 0 0
(integer) 4
127.0.0.1:6379> bitcount hello 0 1
(integer) 8
127.0.0.1:6379> bitcount hello 0 2
(integer) 13
127.0.0.1:6379> bitcount hello 1 1
(integer) 4
127.0.0.1:6379> bitcount hello 1 2
(integer) 9
127.0.0.1:6379> bitcount hello 2 2
(integer) 5
```
### bitop and|or|not|xor destkey key [key...]
- 做多个bitmap的and（交集）、or（并集）、not（非）、xor（异或）操作并将结果保存到destkey中。
```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> set world big
OK
127.0.0.1:6379> bitop and destkey hello world
(integer) 3
127.0.0.1:6379> get destkey
"big"
127.0.0.1:6379> bitop or destkey hello world
(integer) 3
127.0.0.1:6379> get destkey
"big"
127.0.0.1:6379> bitop not destkey hello
(integer) 3
127.0.0.1:6379> get destkey
"\x9d\x96\x98"
127.0.0.1:6379> bitop xor destkey hello world
(integer) 3
127.0.0.1:6379> get destkey
"\x00\x00\x00"
```
### bitpos key targetBit [start] [end] （起始版本：2.8.7）
- 计算位图指定范围（start到end，单位为字节，如果不指定就是获取全部）第一个偏移量对应的值等于targetBit的位置。
```
127.0.0.1:6379> set hello big
OK
127.0.0.1:6379> bitpos hello 1
(integer) 1
127.0.0.1:6379> bitpos hello 0
(integer) 0
127.0.0.1:6379> bitpos hello 1 2 2
(integer) 17
127.0.0.1:6379> bitpos hello 1 2 3
(integer) 17
127.0.0.1:6379> bitpos hello 0 2 3
(integer) 16
127.0.0.1:6379> bitpos hello 0 0 3
(integer) 0
127.0.0.1:6379> bitpos hello 1 0 3
(integer) 1
```

## 场景
### 视频属性的无限延伸
>一个拥有亿级数据量的短视频app，视频存在各种属性(是否加锁、是否特效等等)，需要做各种标记。

使用redis的bitmap进行存储。
key由属性id＋视频分片id组成。value按照视频id对分片范围取模来决定偏移量offset。

伪代码：
```
function set($business_id , $media_id , $switch_status=1){
    $switch_status = $switch_status ? 1 : 0;
    $key = $this->_getKey($business_id, $media_id);
    $offset = $this->_getOffset($media_id);
    return $this->redis->setBit($key, $offse, $switch_status);
}

function get($business_id , $media_id){
    $key = $this->_getKey($business_id,$media_id);
    $offset = $this->_getOffset($media_id);
    return $this->redis->getBit($key , $offset);
}

function _getKey($business_id, $media_id){
        return 'm:'.$business_id.':'.intval($media_id/10000);
}

function _getOffset($media_id){
    return $media_id % 10000;
}

```
### 用户在线状态
>查询用户在线状态

使用bitmap是一个节约空间效率又高的一种方法，只需要一个key，然后用户id为偏移量offset，如果在线就设置为1，不在线就设置为0，3亿用户只需要36MB的空间。
```
$status = 1;
$redis->setBit('online', $uid, $status);
$redis->getBit('online', $uid);
```

### 统计活跃用户
>计算活跃用户的数据情况。

使用时间作为缓存的key，然后用户id为offset，如果当日活跃过就设置为1。之后通过bitOp进行二进制计算算出在某段时间内用户的活跃情况。
```
$status = 1;
$redis->setBit('active_20170708', $uid, $status);
$redis->setBit('active_20170709', $uid, $status);
$redis->bitOp('AND', 'active', 'active_20170708', 'active_20170709'); 
```
### 用户签到
>用户需要进行签到，对于签到的数据需要进行分析与相应的运运营策略。

使用redis的bitmap,由于是长尾的记录，所以key主要由uid组成，设定一个初始时间，往后没加一天即对应value中的offset的位置。

```
$start_date = '20170708';
$end_date = '20170709';
$offset = floor((strtotime($start_date) - strtotime($end_date)) / 86400);
$redis->setBit('sign_123456', $offset, 1);

//算活跃天数
$redis->bitCount('sign_123456', 0, -1)
```