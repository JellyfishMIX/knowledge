## mysql INT和UNSIGNED



## INT(1)和INT(20)

不管你定义 INT (1) 还是 INT (20)，MySQL 在存储的时候都不会做出超出 INT 的限制范围。

那这个括号里面的值是做什么用的呢？

它定义的是显示宽度。

当你在程序中定义显示宽度之后，如果你插入的值不够这个宽度，在查询时会在左边用相应宽度的空格填充。

当你在程序中定义显示宽度之后，如果你插入的值不够这个宽度，在查询时会在左边用相应宽度的空格填充。

这个空格在通常情况下，我们是看不到的，因为在客户端输出时（MySQL Workbench、mysql-connector-java 等客户端）默认会去掉左侧的空格。

所以为什么说它是显示宽度，是因为它并不限制存储的值的范围，不管是 INT (1) 还是 INT (20)，存储范围都是 INT：

- 有符号：-2147483648 ~ 2147483647
- 无符号：0 ~ 4294967295

这里再提及一点，整型类型中还有一个 ZEROFILL 的属性，如果建表时指定了这个属性，刚刚的空格就会变成 0，并且自动给这个列加上 UNSIGNED 属性。

如下：

```mysql
mysql> show create table int_test \G
*************************** 1. row ***************************
       Table: int_test
Create Table: CREATE TABLE `int_test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `col_1` int(1) DEFAULT NULL,
  `col_2` int(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8
1 row in set (0.01 sec)

mysql> alter table int_test modify col_2 int(20) zerofill;
Query OK, 2 rows affected (0.07 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> show create table int_test \G
*************************** 1. row ***************************
       Table: int_test
Create Table: CREATE TABLE `int_test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `col_1` int(1) DEFAULT NULL,
  `col_2` int(20) unsigned zerofill DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> select * from int_test;
+----+-------+----------------------+
| id | col_1 | col_2                |
+----+-------+----------------------+
|  1 |     1 | 00000000000000000001 |
| 10 |    10 | 00000000000000000010 |
+----+-------+----------------------+
2 rows in set (0.01 sec)
```

在上面的实验中我们可以看到，col_2 这一列已经在左侧填充了一串无意义的 0。但是在一些客户端仍然会自动去掉左侧这个无意义的 0。

由于这个显示宽度实在鸡肋，MySQL 官方也表示会在未来的版本中去掉 ZEROFILL 和显示宽度这两个属性。

在现在的版本中，我们只需要了解它，并且不要用错数据类型即可。存储的值的范围不超过 128/256 的话就用 TINYINT，存储值超出 INT 的范围的话就用 BIGINT。

下图是各种整形数据类型的数值范围表：

Required Storage and Range for Integer Types Supported by MySQL

| Type      | Storage(Bytes) | Minimum Value Signed | Maximum Value Signed | Minimum Value Unsigned | Maximum Value Unsigned |
| --------- | -------------- | -------------------- | -------------------- | ---------------------- | ---------------------- |
| TINYINNT  | 1              | -128                 | 127                  | 0                      | 255                    |
| SMALLINT  | 2              | -32768               | 32767                | 0                      | 65535                  |
| MEDIUMINT | 3              | -8388608             | 8388607              | 0                      | 16777215               |
| INT       | 4              | -2147483648          | 2147483647           | 0                      | 4294967295             |
| BIGINT    | 8              | -2^63                | 2^63-1               | 0                      | 2^64-1                 |



## 两个 UNSIGNED 的值无法相减

UNSIGNED 属性就是无符号的数字类型，如果用做自增主键的话，相比有符号的整形数据类型能扩展 1 倍的空间。

有的大型互联网公司会在内部的开发规范中要求使用 UNSIGNED 的自增值。

看起来还蛮不错，但是在使用时有一个小问题你需要注意，那就是两个数值相减得到负数的情形。

我们再来做一个实验，建一个两列都带有 UNSIGNED 属性的表，然后再插入一行数据：

```sql
mysql> create table int_test_2 (col_1 int unsigned, col_2 int unsigned);
Query OK, 0 rows affected (0.03 sec)

mysql> insert into int_test_2 values(1,2);
Query OK, 1 row affected (0.01 sec)

mysql> select col_1 - col_2 from int_test_2;
ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(`imooc_mysql_interview`.`int_test_2`.`col_1` - `imooc_mysql_interview`.`int_test_2`.`col_2`)'
```

在这里你可以看到，本来该输出 - 1 的结果却报错了。并且错误看起来还有点奇怪，提示 BIGINT UNSIGNED 超出了范围。

那为什么会发生这样的问题呢？

其实和编程语言中的问题类似，对于有符号的整形数来说，-1 的十六进制值是 0xFFFFFFFF；而对于无符号的整形数来说，4294967295 的十六进制值也是 0xFFFFFFFF。

在 MySQL 数据库中，**对于 UNSIGNED 数的操作，它的返回值都是 UNSIGNED 的，不能是负值**，所以就导致了上面的错误产生。

那么如果非要获得负值呢？只需要改一下 SQL_MODE 即可：

```sql
mysql> SET sql_mode = 'NO_UNSIGNED_SUBTRACTION';
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> show warnings;
+---------+------+------------------------------------------------------------------------------------------------+
| Level   | Code | Message                                                                                        |
+---------+------+------------------------------------------------------------------------------------------------+
| Warning | 3090 | Changing sql mode 'NO_AUTO_CREATE_USER' is deprecated. It will be removed in a future release. |
+---------+------+------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

但是不太建议这么做，如果需要避免这个情况还是老老实实的使用有符号的整形数据类型，如果担心 INT 的数值范围不够用的话换成 BIGINT 基本也够用了。



## 引用/参考

[MySQL 开发高频面试题精选 - 门牙没了 - 慕课网](https://www.imooc.com/read/88)