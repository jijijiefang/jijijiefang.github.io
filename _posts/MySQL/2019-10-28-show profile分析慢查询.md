---
layout:     post
title:      "Show profile分析慢查询"
date:       2019-10-28 00:00:00
author:     "jiefang"
header-style: text
tags:
    - MySQL
---
# Show profile分析慢查询
## 确定是否支持profile

```
mysql> select @@have_profiling;

+------------------+
| @@have_profiling |
+------------------+
| YES              |
+------------------+

1 row in set, 1 warning (0.00 sec)
```

YES表示支持profile
## 查看profiling是否关闭

```
mysql> select @@profiling;

+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+

1 row in set, 1 warning (0.00 sec)
```

结果为0，profiling默认关闭
## 通过set开启profile

```
mysql> set profiling=1;

Query OK, 0 rows affected, 1 warning (0.00 sec)
```

## 执行SQL语句

```
mysql> select * from t1 where b=1000;
```

## 确定SQL的query id
通过show profiles 语句确定执行过的SQL的query id


```
mysql> show profiles;
+----------+------------+-------------------------------+
| Query_ID | Duration   | Query                         |
+----------+------------+-------------------------------+
|        1 | 0.00063825 | select * from t1 where b=1000 |
+----------+------------+-------------------------------+
1 row in set, 1 warning (0.00 sec)
```

## 查询SQL执行详情
通过show profile for query 可看到执行过的SQL每个状态和消耗时间

```
mysql> show profile for query 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000115 |
| checking permissions | 0.000013 |
| Opening tables       | 0.000027 |
| init                 | 0.000035 |
| System lock          | 0.000017 |
| optimizing           | 0.000016 |
| statistics           | 0.000025 |
| preparing            | 0.000020 |
| executing            | 0.000006 |
| Sending data         | 0.000294 |
| end                  | 0.000009 |
| query end            | 0.000012 |
| closing tables       | 0.000011 |
| freeing items        | 0.000024 |
| cleaning up          | 0.000016 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)
```


