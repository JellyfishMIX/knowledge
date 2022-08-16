# mysql 常用 sql



## 查询所有数据库

```mysql
show databases;
```



## 切换选中的数据库

假设有一个数据库叫 o2o

```mysql
use o2o;
```



## 查询数据库中所有表

需要先选中某个数据库

```mysql
show tables;
```



## 查询某张完整的表

假设某张表叫 tb_area

```mysql
select * from tb_area;
```



##  查询指定步长

从起始时间2020-05-20 21:03:00查询间隔1秒到结束时间2020-05-20 21:04:00的所有时间
即：从起始时间2020-05-20 21:03:00以1秒为步长的到到结束时间2020-05-20 21:04:00的所有时间

```
select * from wx_interchange.file_info
where (datediff('2020-05-20 21:03:00', wx_interchange.file_info.creation_time)%1) = 0
  and wx_interchange.file_info.creation_time >= '2020-05-20 21:03:00'
  and wx_interchange.file_info.creation_time <= '2020-05-20 21:04:00';
```



## 查询某字段值重复的数据，及其重复次数

查询user表中，user_name字段值重复的数据及重复次数

```html
select user_name,count(*) as count from user group by user_name having count>1;
```



## mysql 表新增一列

有这样的需求，已经建立的表，随着需求的变化，会需要在这个表增加一列。当然

可以新建表建立联系满足需求。

但就仅新增一列这个问题，可以有以下操作：

```sql
ALTER ...  ADD COLUMN .... 
```

在表的最后一列增加新的一列

```sql
ALTER TABLE `tbname`
	ADD COLUMN `state` TINYINT(2) NOT NULL DEFAULT '0' COMMENT '0为添加1为编辑';
```

在指定的位置增加新的一列

```sql
ALTER TABLE `tbname`
	ADD COLUMN `state` TINYINT(2) NOT NULL DEFAULT '0' COMMENT '0为添加1为编辑' AFTER `column_name`;
```

在第一列增加新的一列

```sql
ALTER TABLE `tbname`
	ADD COLUMN `state` TINYINT(2) NOT NULL DEFAULT '0' COMMENT '0为添加1为编辑' FIRST;
```



## 表修改 DDL

```sql
alter table `tbname` modify column `column_name` int not null default 0  comment '注释';
```



## 表修改列名

```sql
alter table `tbname` change `column1` `column2` int not null default 0  comment '注释';
```



## 引用/参考

[mysql表新增添加一列 - Agly_Charlie - CSDN](https://blog.csdn.net/Agly_Clarlie/article/details/78195162)