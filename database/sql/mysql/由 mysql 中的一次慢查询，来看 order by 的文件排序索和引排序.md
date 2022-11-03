# 由 mysql 中的一次慢查询，来看 order by 的文件排序和索引排序



## 说明

1. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 慢查询

笔者在之前工作期间，写过一个列表的查询需求，引发了线上的慢查询问题（一般情况下，我们把查询时间超过 1s 的查询称为慢查询，不同团队定义不同）。

数据表量级大概在 3800w 行。敏感信息已屏蔽，且字段名非真实字段名，能表达意思即可。我们用简单的字段来复现问题场景。

content 要查寻的内容

type 类型，可以理解为 where 中的查询条件

create_time 创建时间

audit_time 审核时间

引发慢查询的 sql 如下：

```sql
select content from demo_table where create_time >= ${startTime} and create time < ${endtTime} and type = 2 order by audit_time desc;
```

请注意，${} 这里表示是一个变量，执行 sql 的时候会替换成实际的值。另外这是示意 sql，不要考虑 sql 注入风险等问题，实际的 sql 比这个严谨。

create_time 加了索引，audit_time 是没有索引的，order by audit_time 最后走了文件排序，这个文件排序导致了慢查询。

解决方案是，排序字段换成了使用了索引的 create_time 字段，利用索引的有序性，避免走文件排序。



## 文件排序 filesort

filesort 并不是说通过磁盘文件进行排序，只是告诉我们进行了一个排序操作。即在 MySQL Query Optimizer 所给出的执行计划（通过 EXPLAIN 命令查看）中被称为 filesort

文件排序是通过相应的排序算法，将取得的数据在内存中进行排序。mysql 需要将数据在内存中进行排序，所使用的内存区域是通过 sort_buffer_size 系统变量所设置的排序区。这个排序区是每个 thread 独享的，所以说可能在同一时刻在MySQL 中可能存在多个 sort buffer 内存区域。

在 mysql 中 filesort 的实现算法有两种：

1. 双路排序：根据查询条件取出排序字段，和可以直接定位行数据的行指针信息，这也是第一次 IO。然后在 sort buffer 中进行排序。把行指针排序好后，要再进行一次 IO 将具体的行查询字段读出，才能返回结果。

2. 单路排序：根据查询条件取出排序字段和需要查询的行字段（一次 IO），然后在 sort buffer 中进行排序。排序后就可以返回结果了。

在 mysql 4.1 版本之前只有双路排序，单路排序是从 mysql 4.1 开始推出的改进算法。单路排序的主要目的，为了把双路排序中需要两次 IO，变成只需要一次 IO，但相应也会耗用更多的 sortbuffer 空间。mysql 4.1 及之后的版本同时支持双路排序与单路排序。

mysql 主要通过比较我们所设定的系统参数 max_length_for_sort_data 的大小，和 query 语句所需取出的字段类型大小总和，来判断使用哪一种排序算法。如果 max_length_for_sort_data 足够大，则使用速度更快的单路排序，反之使用所需内存小的双路排序。所以如果想优化文件排序的速度，需要注意 max_length_for_sort_data 参数的设置。

当表中数据量非常大时，更推荐的做法是，不要使用文件排序，给 order by 的排序字段加索引。



## 索引排序

在使用 explain 分析查询的时候，利用有序索引获取有序数据显示 Using index，而文件排序显示 Using filesort

InnoDB 中的索引是 B+ 树结构，B+ 树中的索引是有序的。因此可以通过给排序字段加索引，利用索引的有序性，来加快排序速度。



## 应注意的

当表中数据量很大时，应当注意 order by 的文件排序的时间消耗，预估到 order by 可能造成的慢查询风险，对 order by 的条件字段加索引，利用索引的有序性来避免文件排序。

慢查询棘手的地方在于测试环节很难发现，因为测试环境的数据量通常比生产环境的数据量级少很多。在数据量不大的情况下，慢查询问题很难通过测试发现。