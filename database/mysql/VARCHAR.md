# VARCHAR



## CHAR (50) 和 VARCHAR (50) 有什么区别？

刚工作不久的同学可能会有这个疑问，为什么大家都喜欢用 VARCHAR，CHAR 却很少见，存的长度不是都一样吗？

首先要说明的一点，**CHAR 和 VARCHAR 在存储方式上存在着差异**：CHAR 是定长字符，MySQL 数据库会根据建表时定义的长度给它分配相应的存储空间。而 VARCHAR 是可变长度字符的数据类型，在存储时只使用必要的空间。

举个例子，假如一张表上有两列，分别是 CHAR (20) 和 VARCHAR (20)，我们插入一个字符串 “abcd”，在数据库中存储时，CHAR 会使用全部的 20 个字符的长度，不足的部分用空格填充，而 VARCHAR 仅仅就只使用 4 个字符的长度。

其次，由于 **CHAR 数据类型的这个特性，在将数据写入表中时，如果字符串尾部存在空格，会被自动删除**，而 VARCHAR 数据类型会保留这个空格。在一些特殊场景中要注意这个问题。所以推荐你使用 CHAR 数据类型存储一些固定长度的字符串，比如身份证号、手机号、性别等。

最后，**CHAR 和 VARCHAR 的存储长度不同**。CHAR 数据类型可定义的最大长度是 255 个字符，而 VARCHAR 根据所使用的字符集不同，最大可以使用 65535 个字节。注意我刚说的 VARCHAR 的最大长度不是字符数而是字节数，那么新的问题来了，我们接着往下看。



## VARCHAR 能使用的最大长度是多少？

由于 VARCHAR 能存储的最大长度会因为你在表定义中使用的字符集不同而发生变化，下面我们就以业内使用较多的 UTF8 这个字符集作为前提条件来做个分析。

我们再看一个例子：

```sql
mysql> create table varchar_test2(col_1 varchar(65535))charset=utf8 engine=innodb;
ERROR 1074 (42000): Column length too big for column 'col_1' (max = 21845); use BLOB or TEXT instead

mysql> create table varchar_test2(col_1 varchar(21845))charset=utf8 engine=innodb;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

mysql> create table varchar_test2(col_1 varchar(21844))charset=utf8 engine=innodb;
Query OK, 0 rows affected (0.02 sec)
```

由于 UTF8 字符集中一个汉字占用 3 个字节，因此我们能创建的最大长度理论上应该是 21845（65535/3=21845）。但是为什么 varchar (21845) 仍然报错，而使用 varchar (21844) 却创建成功？

这是因为触及了 MySQL 数据库定义的 VARCHAR 的最大行长度限制。

虽然 MySQL 官方定义了最大行长度是 65535 个字节，但是因为还有别的开销，我们能使用的最大行长度只有 65532。

刚刚的实验中，我们把 VARCHAR 的字段长度改成 21844 后腾出来 3 个字节 (65535-21844*3=3)，因此可以创建成功。

因此**在使用了 UTF-8 的字符集时，VARCHAR 的最大长度为 21844**。

另外提醒你注意一下，做表设计时不要肆意的放飞自我，在单表上设计出一堆较大的 VARCHAR 字段，在失去了扩展性时以后可能会哭。因为 MySQL 的最大行长度限制不只是 1 个 VARCHAR 列，而是所有列的长度总和。

我们再做个实验观察一下：

```sql
mysql> create table varchar_test3(id int auto_increment, col_2 varchar(21844), primary key(id))charset=utf8 engine=innodb;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

mysql> create table varchar_test3(id int auto_increment, col_2 varchar(21843), primary key(id))charset=utf8 engine=innodb;
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs

mysql> create table varchar_test3(id int auto_increment, col_2 varchar(21842), primary key(id))charset=utf8 engine=innodb;
Query OK, 0 rows affected (0.02 sec)

mysql> create table varchar_test4(id int auto_increment, col_2 varchar(21842), col_3 smallint, primary key(id))charset=utf8 engine=innodb;
Query OK, 0 rows affected (0.02 sec)
```

你可以自行计算一下，在上面的这个实验中，所有列的字节数加起来也是不能超过 65532 的，超出时会报错。

你可能还有一个疑惑，在 UTF8 中英文字符不是只占用 1 个字节吗，那 varchar (21844) 使用了 65532 个字节，如果我只存英文字符的话是不是就能存 65532 个？

然而并非如此，VARCHAR (M) 中的 M 仍然表示的是 M 个字符，而不是 M 个字节。

只不过它在存储的时候仍然是按实际字节数来存的。所以在 UTF8 的字符编码下，我们能使用的最大长度就只有 21844 个 VARCHAR 字符。

如果要存储更多的字符该怎么办呢？使用 TEXT、BLOB 这样的大对象列类型。因为**这些大对象可以把数据存放到溢出页面上**，也就是 DBA 们常说的行溢出。



## VARCHAR 数据类型优化

### 只分配所需要用的 VARCHAR 空间

虽然使用 VARCHAR (50) 和 VARCHAR (1000) 存储‘abcd’的存储空间开销是一样的，但是当你在读取数据时，把这些数据读取到内存的过程中，MySQL 数据库需要分配相应大小的内存空间来存放数据。

所以更大的 VARCHAR 列在读取时要使用更大的内存空间，即使它实际上只存储了一丁点数据。

并且在操作这个表的过程中，如果遇到一些聚合（GROUP BY）或排序（ORDER BY）的操作，需要调用内存临时表或磁盘临时表时，性能会更加糟糕。

因此，**在保留一定冗余的前提下，只给 VARCHAR 分配恰到好处的空间**使用。

### VARCHAR 的字段过长会导致行溢出

你在给 MySQL 的数据表加索引时，可能遇到过要在大的 VARCHAR 字段上创建索引却发现只能创建前缀索引的问题。那这个其实是和行溢出有关。

```sql
mysql> create index idx_col_2 on varchar_test4(col_2);
ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes

mysql> create index idx_col_2 on varchar_test4(col_2(3072));
ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes

mysql> create index idx_col_2 on varchar_test4(col_2(1024));
Query OK, 0 rows affected (0.02 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> drop index idx_col_2 on varchar_test4;
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> create index idx_col_2 on varchar_test4(col_2(1025));
ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes
```

上面实验可以看到，**最大的索引长度不能超过 3072 字节**（在 UTF-8 的字符集中对应的是 1024 个字符）。

由于行溢出是另外一个话题，这里就不过多赘述，我们只说说大字段和行溢出造成的性能问题。

1. 大字段会占用较大的内存，使得 MySQL 内存利用率较差
2. 行溢出的数据在读取时需要多一个 IO，造成 IO 效率下降
3. 行溢出会使用 uncompress BLOB page，造成 InnoDB 的表空间越来越大
4. InnoDB 中的大字段在做更新和删除操作时，只能进行悲观操作，这会造成并发性能下降。

另外，因为 InnoDB 的数据页默认是 16K，每个页中至少存放 2 行数据，因此建议 VARCHAR 字段的总长度不要超过 8K。



## 引用/参考

[MySQL 开发高频面试题精选 - 门牙没了 - 慕课网](https://www.imooc.com/read/88)