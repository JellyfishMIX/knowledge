# 根据 mysql 索引的区分度来判断是否要建立索引



## 问题

我们现在有一张表：

```sql
create table `award`
(
    `id`          bigint unsigned not null auto_increment comment '主键id',
    `book_id`     varchar(20)     not null comment '图书编号',
    `author_id`   varchar(20)     not null comment '作者编号',
    `cup_type`    int             not null comment '奖项类型，1金，2银，3铜',
    `cup_time`    datetime        not null comment '获奖日期',
    `create_time` timestamp       not null default current_timestamp comment '创建时间，自动写入',
    `update_time` timestamp       not null default current_timestamp on update current_timestamp comment '更新时间，自动写入',
    primary key `idx_id` (id)
) engine = InnoDB
  default charset = utf8mb4 comment '奖项表';
```

```sql
# 查询获得过金奖的图书有多少本，银奖的有多少本？
# 获得过金奖的图书数量
select count(1)
from award
where cup_type = 1;
# 获得过银奖的图书数量
select count(1)
from award
where cup_type = 2;
```

针对这两个 sql 查询，要不要建立索引？



## 分析

我们知道 mysql 建立索引时，通常有一条规范：不要给区分度低的字段建立索引。

意思是如果一个字段有很多相同的值，建议不要给它建立索引。

为什么会有这样的规定呢？我们结合 B+ 树的结构来看一下：

![image-20220728221436923](https://image-hosting.jellyfishmix.com/20220728221507.png)

建立索引，本来是想靠树的结构加快查询速度，如果这个索引字段有很多相同的值，会导致通过索引定位后，得到的结果集很多

。比如我们有 100w 行数据，其中性别字段 50w 是男，50w 是女，这种时候给性别字段加索引没有意义，因为通过索引定位到的结果集是 50w...这结果集还是很大，并没有加快查询速度。



## 进一步的思考

那是不是对于取值只有几种类型(比如取值只有1,2,3,4)的字段，我们就不应该建立索引了呢？

并不是，我们来看下面描述的场景：

有一张大表（3800w+行），有一个 check_type 字段，大概对应待审核、待确认、已完成、未通过这几项。待审核、待确认、未通过的比例小(占比3%以下)，此时查找 待审核、待确认、未通过 的行，走 check_type 索引查会比较快（因为这三种值所占比例小，通过索引查询这三种值得到的结果集小）。

而恰好这三个 check_type 是最常被查询的，这样在频繁的查询中，很大比例的查询都利用索引快速找到了结果。

对于取值只有几种类型(比如取值只有1,2,3,4)的字段，我们要结合查询频繁的 sql、查询的值对应的数据比例，来判断建立索引的性价比。如果性比价高就可以建立索引。