# InsertBuffer和ChangeBuffer
## InsertBuffer
对于非聚集索引的插入时，先判断插入的非聚集索引页是否在缓冲池中。如果在，则直接插入；
如果不在，则先放入 Insert Buffer 中，然后再以一定频率和情况进行 Insert Buffer 和辅助索
引页子节点的 merge 操作。这时通常能将多个插入合并到一个操作中（因为在一个索引页
中），就大大提高了非聚集索引的插入性能。
```
为什么要增加 Insert Buffer？
增加 Insert Buffer 有两个好处：
减少磁盘的离散读取
将多次插入合并为一次操作
```
但是得注意的是，使用 Insert Buffer 得满足两个条件：
- 索引是辅助索引
- 索引不是唯一
## Change Buffer
InnoDB 从 1.0.x 版本开始引入了 Change Buffer，可以算是对 Insert Buffer 的升级。从这个
版本开始，InnoDB 存储引擎可以对 insert、delete、update 都进行缓存。
影响参数有两个：
- **innodb_change_buffering**：确定哪些场景使用 Change Buffer，它的值包含：none、inserts、deletes、changes、purges、all。默认为 all，表示启用所有。
- **innodb_change_buffer_max_size**：控制 Change Buffer 最大使用内存占总 buffer pool
的百分比。默认25，表示最多可以使用 buffer pool 的 25%，最大值50。

跟 Insert Buffer 一样，Change Buffer 也得满足这两个条件：
- 索引是辅助索引
- 索引不是唯一

为什么唯一索引的更新不使用 Change Buffer ?
原因：唯一索引必须要将数据页读入内存才能判断是否违反唯一性约束。如果都已经读入到内存
了，那直接更新内存会更快，就没必要使用 Change Buffer 了。