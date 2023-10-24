---
layout: mysql
title: mysql
date: 2023-10-24 14:15:27
tags:
---

### mysql性能优化
#### 执行计划的含义
``` 
select_type

SIMPLE 简单的查询 不包含子查询 和 union
PRIMARY 如果查询有任何复杂的字部分，则最外层标记为PRIMARY
SUBQUERY 子查询中的SELECT（换句话说，不在FROM子句中的），标记为SUBQUERY
DERIVED  DERIVED 值用来表示包含在FROM字句的子查询中的SELECT
UNION 在union中的第二个和随后的SELECT被标记为union
UNION RESULT 用来从union的匿名临时表检索结果的SELECT被标记为UNION RESULT
```
``` 
table 列
这一列显示了对应行正在访问那个表 
```
``` 
type列
ALL 全表扫描
INDEX 这个跟全表扫描类似，只是mysql是安装索引次序扫描而不是行，主要优点是避免了排序，最大的缺点是承担了按索引读取整个表的开销，若是按随机次序访问行，开销将会非常大，如果Extra列显示 Using Index 说明使用了覆盖索引，而不用回表，比按所以次序全表扫描开销要小很多

```