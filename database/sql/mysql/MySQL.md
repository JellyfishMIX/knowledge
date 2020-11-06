# MySQL



## 索引

### 组合索引

![img](https://uploadfiles.nowcoder.com/images/20170815/5994168_1502785746568_3872455FABFD1B29AF3688B49786FD79)

- MySQL组合索引（复合索引）的最左优先原则。最左优先就是说组合索引的第一个字段必须出现在查询组句中，这个索引才会被用到。只要组合索引最左边第一个字段出现在Where中，那么不管后面的字段出现与否或者出现顺序如何，MySQL引擎都会自动调用索引来优化查询效率。
- 根据最左匹配原则可以知道B-Tree建立索引的过程，比如假设有一个3列索引(`col1`, `col2`, `col3`)，那么MySQL只会会建立三个索引(`col1`)，(`col1`, `col2`)， (`col1`, `col2`, `col3`)。
- 题目会创建三个索引：(`plat_order_id`)，(`plat_order_id`,  `plat_game_id`)，(`plat_order_id`,  `plat_game_id`, `plat_id`)。
- 根据最左匹配原则，where语句必须要有`plat_order_id`才能调用索引（如果没有`plat_order_id`那么一个索引也调用不到），如果同时出现`plat_order_id`与`plat_game_id`则会调用两者的组合索引，如果同时出现三者则调用三者的组合索引。
- 如出现了(`col1`, `col3`)，只使用了第一个索引(`col1`)。



## 数据类型

### 为什么mysql的数值不应设置为null，而是not null

- B树索引是不会存储NULL值的，所以如果索引的字段可以为NULL，则此字段无法使用索引。