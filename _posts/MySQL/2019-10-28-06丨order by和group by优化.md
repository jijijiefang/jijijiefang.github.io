---
layout:     post
title:      "06丨order by和group by优化"
date:       2019-10-28 00:00:05
author:     "jiefang"
header-style: text
tags:
    - MySQL
---
# order by和group by优化

## order by

### MySQL 的排序方式
按照排序原理分，MySQL 排序方式分两种：
- 通过有序索引直接返回有序数据
- 通过 Filesort 进行的排序

### Filesort 是在内存中还是在磁盘中完成排序的？

MySQL 中的 Filesort 并不一定是在磁盘文件中进行排序的，也有可能在内存中排序，内存排序
还是磁盘排序取决于排序的数据大小和 `sort_buffer_size` 配置的大小。
- 如果 “排序的数据大小” < `sort_buffer_size`: 内存排序
- 如果 “排序的数据大小” > `sort_buffer_size`: 磁盘排序

```
"filesort_summary": {
              "rows": 10000,
              "examined_rows": 10000,
              "number_of_tmp_files": 2,
              "sort_buffer_size": 262144,
              "sort_mode": "<sort_key, additional_fields>"
            } /* filesort_summary */
```
- rows：预计扫描的行数
- examined_rows：参与排序的行
- number_of_tmp_files：使用临时文件的个数
- sort_buffer_size：sort_buffer 的大小
- sort_mode：排序模式

number_of_tmp_files等于7，所以表示使用的是磁盘排序。对于number_of_tmp_files 等于 7 表示该 SQL 将需要排序的数据分为 7 份，然后每份单独排序，再存放在 7 个临时文件中，最后把7个临时文件合并成一个大的有序文件
### Filesort 下的排序模式
Filesort 下的排序模式有三种，具体介绍如下：
- < sort_key, rowid>双路排序（又叫回表排序模式）：是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sortbuffer中进行排序，排序完后需要再次取回其它需要的字段；
- < sort_key, additional_fields>单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；
- < sort_key, packed_additional_fields>打包数据排序模式：与单路排序相似，区别是将char 和 varchar 字段存到 sort buffer 中时，更加紧缩。

MySQL 通过比较系统变量`max_length_for_sort_data`的大小和需要查询的字段总大小来判断使用哪种排序模式。
- 如果 `max_length_for_sort_data` 比查询字段的总长度大，那么使用 < sort_key,
additional_fields >排序模式；
- 如果 `max_length_for_sort_data` 比查询字段的总长度小，那么使用 <sort_key,
  rowid> 排序模式。 

```
 select a,c,d from t1 where a=1000 order by d;
```

#### 单路排序的详细过程：
 
1. 从索引 a 找到第一个满足 a = 1000 条件的主键 id
2. 根据主键 id 取出整行，**取出 a、c、d 三个字段的值，存入 sort_buffer 中**
3. 从索引 a 找到下一个满足 a = 1000 条件的主键 id
4. 重复步骤 2、3 直到不满足 a = 1000
5. 对 sort_buffer 中的数据按照字段 d 进行排序
6. 返回结果给客户端

#### 双路排序的详细过程：

1. 从索引a找出第一个满足a=1000的主键id
2. 根据主键id取出整行，**把排序字段d和主键id这两个字段放到sort_buffer中**
3. 从索引 a 取下一个满足 a = 1000 记录的主键 id
4. 重复 3、4 直到不满足 a = 1000
5. 对 sort_buffer 中的字段 d 和主键 id 按照字段 d 进行排序
6. 遍历排序好的 id 和字段 d，按照 id 的值回到原表中取出 a、c、d三个字段的值返回给客户端

其实对比两个排序模式，单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键和需要排序的字段放到 sort buffer中进行排序，然后再通过主键回到原表查询需要的字段。

如 果 MySQL 排 序 内 存 配 置 的 比 较 小 并 且 没 有 条 件 继 续 增 加 了 ， 可 以 适 当 把max_length_for_sort_data 配 置 小 点 ， 让 优 化 器 选 择 使 用 rowid 排 序 算 法 ， 可 以在sort_buffer中一次排序更多的行，只是需要再根据主键回到原表取数据。如果 MySQL排序内存有条件可以配置比较大，可以适当增大max_length_for_sort_data的值，让优化器优先选择全字段排序，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。

所以 MySQL 通过 max_length_for_sort_data这个参数来控制排序，在不同场景使用不同的排序模式，从而提升排序效率。
## order by优化
### 1、添加索引
#### 排序字段添加索引
在排序字段上添加索引来优化排序语句
#### 多字段排序优化
多个字段排序，在多个字段上添加联合索引来优化排序，注意排序字段顺序与索引中字段顺序一致。
#### 等值查询排序优化
对于等值查询再排序的语句，可以通过条件字段和排序字段添加联合索引来优化此类排序。
###  2、去掉不必要的返回字段
查询所有字段不走索引的原因是：扫描整个索引并查找没索引的行的成本比扫描全表的成本更高，优化器放弃使用索引。

###  3、修改参数
两个跟排序有关的参数 ：**max_length_for_sort_data**、**sort_buffer_size**。
- `max_length_for_sort_data`：如果觉得排序效率比较低，可以适当加大
max_length_for_sort_data的值，让优化器优先选择全字段排序。当然不能设置过大，可能会导致 CPU 利用率过低或者磁盘 I/O 过高；
- `sort_buffer_size`：适当加大sort_buffer_size的值，尽可能让排序在内存中完成。但不能设置过大，可能导致数据库服务器 SWAP。

### 4、无法利用索引排序的情况
#### 使用范围查询再排序
联合索引中前面的字段使用了范围查询，对后面的字段排序使用不了索引排序。
#### ASC和DESC混合使用无法索引
联合索引多个字段排序时，如果一个是顺序，一个是倒序，则使用不了索引。

## 3、group by 优化
默认情况下，会对gorup by字段排序，优化方式与order by基本一致，如果只分组不排序，可以指定 order by null禁止排序。