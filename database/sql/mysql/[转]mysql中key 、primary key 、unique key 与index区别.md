# [转]mysql中key 、primary key 、unique key 与index区别



**索引**被用来快速找出在一个列上用一特定值的行。没有索引，MySQL不得不首先以第一条记录开始并然后读完整个表直到它找出相关的行。

表越大，花费时间越多。如果表对于查询的列有一个索引，MySQL能快速到达一个位置去搜寻到数据文件的中间，没有必要考虑所有数据。

如果一个表有1000行，这比顺序读取至少快100倍。注意你需要存取几乎所有1000行，它较快的顺序读取，因为此时我们避免磁盘寻道。 

所有的MySQL**索引**(PRIMARY、UNIQUE和INDEX)**在B树中存储**。字符串是自动地压缩前缀和结尾空间。

**索引用于：** 

快速找出匹配一个WHERE子句的行； 
当执行联结时，从其他表检索行； 
对特定的索引列找出MAX()或MIN()值； 
如果排序或分组在一个可用键的最左面前缀上进行(例如，ORDER BY key_part_1,key_part_2)，排序或分组一个表。

如果所有键值部分跟随DESC，键以倒序被读取。 
在一些情况中，一个查询能被优化来检索值，不用咨询数据文件。

如果对某些表的所有使用的列是数字型的并且构成某些键的最左面前缀，为了更快，值可以从索引树被检索出来。 

—————————————————————————————————————————————————————————————————————————————

下面是建表的语句:

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE `phpcolor_ad` (  
`id` mediumint(8) NOT NULL AUTO_INCREMENT,  
`name` varchar(30) NOT NULL,  
`type` mediumint(1) NOT NULL,  
`code` text,  
PRIMARY KEY (`id`),  
KEY `type` (`type`)  
);  
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

 

最后一句的KEY `type` (`type`)是什么意思？

**如果只是key的话，就是普通索引**。

​     mysql的key和index多少有点令人迷惑，单独的key和其它关键词结合的key(primary key)实际表示的意义是不同，这实际上考察对数据库体系结构的了解的。
1 ：

key 是数据库的**物理结构**，它包含两层意义和作用，

**一是约束（偏重于约束和规范数据库的结构完整性），**

**二是索引（辅助查询用的）。**

包括primary key, unique key, foreign key 等。

**primary key** 有两个作用，一是约束作用（constraint），用来**规范一个存储主键和唯一性**，但同时也在此key上建立了一个主键索引；  

​             PRIMARY KEY 约束：唯一标识数据库表中的每条记录；

​                                 主键必须包含唯一的值；

​                                 主键列不能包含 NULL 值；

​                                 每个表都应该有一个主键，并且每个表只能有一个主键。（PRIMARY KEY 拥有自动定义的 UNIQUE 约束）

**unique key** 也有两个作用，一是约束作用（constraint），规范数据的**唯一性**，但同时也在这个key上建立了一个唯一索引；

UNIQUE 约束：唯一标识数据库表中的每条记录。
                          UNIQUE 和 PRIMARY KEY 约束均为列或列集合提供了**唯一性**的保证。
                          （每个表可以有多个 UNIQUE 约束，但是每个表只能有一个 PRIMARY KEY 约束）

**foreign key** 也有两个作用，一是约束作用（constraint），规范数据的引用完整性，但同时也在这个key上建立了一个index；

 

可见，mysql的key是同时具有constraint和index的意义，这点和其他数据库表现的可能有区别。

（至少在Oracle上建立外键，不会自动建立index），因此**创建key**也有如下几种方式：
（1）在**字段级**以**key**方式建立， 如 create table t (id int not null primary key);
（2）在**表级**以**constraint**方式建立，如create table t(id int, CONSTRAINT pk_t_id PRIMARY key (id));
（3）在**表级**以**key**方式建立，如create table t(id int, primary key (id));

其它key创建类似，但不管那种方式，既建立了constraint，又建立了index，只不过index使用的就是这个constraint或key。

 

2： index是数据库的物理结构，它只是辅助查询的，它创建时会在另外的表空间（mysql中的innodb表空间）以一个类似目录的结构存储。索引要分类的话，分为前缀索引、全文本索引等；
    因此，索引只是索引，**它不会去约束索引的字段的行为**（那是key要做的事情）。如，create table t(id int,**index inx_tx_id (id)**);


3 总结，最后的释疑：
（1）我们说**索引分类**，分为

主键索引（必须指定为“PRIMARY KEY”，没有PRIMARY Index）、

唯一索引（unique index，一般写成unique key）、

普通索引(index，只有这一种才是纯粹的index)等，**也是基于是不是把index看作了key**。
      比如 create table t(id int, **unique index**inx_tx_id (id));**--index当作了key使用**

（2）最重要的也就是，不管如何描述，需要理解index是纯粹的index（普通的key，或者普通索引index），还是被当作key（如：unique index、unique key和primary key），若当作key时则会有两种意义或起两种作用。

—————————————————————————————————————————————————————————————————————————————

MySQL Key值（PRI, UNI, MUL）的含义：

PRI主键约束；

UNI唯一约束；

MUL可以重复。

注：若是普通的key或者普通的index（实际上，普通的key与普通的index同义）。

 

当我们在desc 表名; 的时候,有一个Key值，表示该列是否含有索引
假设表结构如下所示
mysql> desc aa;
+-------+---------+------+-----+---------+-------+
| Field | Type  | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| id  | int(11) | YES |   | NULL  |    |
+-------+---------+------+-----+---------+-------+
| xx  | int(11) | YES | PRI | NULL  |    |
+-------+---------+------+-----+---------+-------+
| yy  | int(11) | YES | UNI | NULL  |    |
+-------+---------+------+-----+---------+-------+
| zz  | int(11) | YES | MUL | NULL  |    |
+-------+---------+------+-----+---------+-------+
1 row in set (0.00 sec)

我们看到Key那一栏，可能会有4种值，即'啥也没有','PRI','UNI','MUL'
\1. 如果Key是空的, 那么该列值的可以重复，表示该列没有索引, 或者是一个**非唯一**的复合索引的**非**前导列
\2. 如果Key是PRI, 那么该列是主键的组成部分
\3. 如果Key是UNI, 那么该列是一个唯一值索引的第一列(前导列)，且不能含有空值(NULL)
\4. 如果Key是MUL, 那么该列的值可以重复, 该列是一个**非唯一**索引的前导列(第一列)或者是一个唯一性索引的组成部分但是可以含有空值NULL

**注：**
1、如果对于一个列的定义，同时满足上述4种情况的多种，比如一个列既是PRI，又是UNI（如果是PRI，则一定是UNI）
那么"desc 表名"; 的时候，显示的Key值按照优先级来显示 **PRI->UNI->MUL**
那么此时，显示PRI。

2、如果某列不能含有空值，同时该表没有主键，则一个唯一性索引列可以显示为PRI，

3、如果多列构成了一个唯一性复合索引，那么一个唯一性索引列可以显示为MUL。（因为虽然索引的多列组合是唯一的，比如ID+NAME是唯一的，但是每一个单独的列依然可以有重复的值，因为只要ID+NAME是唯一的即可）

 

 

**一、key与primary key区别** 

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE wh_logrecord (   
logrecord_id int(11) NOT NULL auto_increment,   
user_name varchar(100) default NULL,   
operation_time datetime default NULL,   
logrecord_operation varchar(100) default NULL,   
PRIMARY KEY (logrecord_id),   
KEY wh_logrecord_user_name (user_name)   
)；
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

解析： 
KEY wh_logrecord_user_name (user_name) 
**本表的user_name字段与wh_logrecord_user_name表user_name字段建立外键** 
括号外是建立外键的对应表，括号内是对应字段 
类似还有 KEY user(userid) 
当然，key未必都是外键 

总结： 
Key是**索引约束**，对表中字段进行约束索引的，都是通过primary foreign unique等创建的。常见有foreign key，外键关联用的。 

KEY forum (status,type,displayorder) # 是多列索引（键） 
KEY tid (tid)             # 是单列索引（键）。 

如建表时： KEY forum (status,type,displayorder) 
select * from table group by status,type,displayorder 是否就自动用上了此索引， 
而当 select * from table group by status 此索引有用吗？ 

key的用途：**主要是用来加快查询速度的。** 

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE `admin_role` (  
  `adminSet_id` varchar(32) NOT NULL,  
  `roleSet_id` varchar(32) NOT NULL,  
  PRIMARY KEY (`adminSet_id`,`roleSet_id`),  
  KEY `FK9FC63FA6DAED032` (`adminSet_id`),  
  KEY `FK9FC63FA6C7B24C48` (`roleSet_id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8;  
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

 

主键，两个列组合在一起，是唯一的，内建唯一性索引，并且不能为NULL
另外，两个Key定义，相当于分别对这两列建立索引。

innodb
primary key  主键聚集索引
key 普通索引

 

**二、KEY与INDEX区别** 



KEY通常是INDEX同义词。如果关键字属性PRIMARY KEY在列定义中已给定，则PRIMARY KEY也可以只指定为KEY。

这么做的目的是与其它数据库系统兼容。

PRIMARY KEY是一个唯一KEY，此时，所有的关键字列必须定义为NOT NULL。

如果这些列没有被明确地定义为NOT NULL，MySQL应隐含地定义这些列。一个表只有一个PRIMARY KEY。 


MySQL 中Index 与Key 的区别 

Key即键值，是关系模型理论中的一部份，比如有主键（Primary Key)，外键（Foreign Key）等，用于数据完整性检否与唯一性约束等。

而Index则处于**实现层面**，比如可以**对表的任意列建立索引**，那么当建立索引的列处于SQL语句中的Where条件中时，就可以得到快速的数据定位，从而快速检索。

至于Unique Index，则只是属于Index中的一种而已，建立了Unique Index表示此列数据不可重复，猜想MySQL对Unique Index类型的索引可以做进一步特殊优化吧。 


于是乎，在设计表的时候，Key只是要处于模型层面的，而当需要进行查询优化，则对相关列建立索引即可。 

另外，在MySQL中，对于一个Primary Key的列，MySQL已经自动对其建立了Unique Index，无需重复再在上面建立索引了。 

搜索到的一段解释： 

  Note that “primary” is called PRIMARY KEY not INDEX. 
  KEY is something on the logical level, describes your table and database design (i.e. enforces referential integrity …) 
  INDEX is something on the physical level, helps improve access time for table operations. 
  Behind every PK there is (usually) unique index created (automatically). 

**三、mysql中UNIQUE KEY和PRIMARY KEY有什么区别** 

1，Primary key的1个或多个列必须为NOT NULL，如果列为NULL，在增加PRIMARY KEY时，列自动更改为NOT NULL。

而UNIQUE KEY 对列没有此要求 

2，一个表**只能有一个PRIMARY KEY**，但可以有多个UNIQUE KEY 

3，主键和唯一键约束是通过参考索引实施的，如果插入的值均为NULL，

则根据索引的原理，全NULL值不被记录在索引上，所以插入全NULL值时，可以有重复的，而其他的则不能插入重复值。 

alter table t add constraint uk_t_1 unique (a,b); 
insert into t (a ,b ) values (null,1);  # 不能重复 
insert into t (a ,b ) values (null,null);#可以重复 

**四、使用UNIQUE KEY** 

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE `secure_vulnerability_warning` (   
  `id` int(10) NOT NULL auto_increment,   
  `date` date NOT NULL,   
  `type` varchar(100) NOT NULL,   
  `sub_type` varchar(100) NOT NULL,   
  `domain_name` varchar(128) NOT NULL,   
  `url` text NOT NULL,   
  `parameters` text NOT NULL,   
  `hash` varchar(100) NOT NULL,   
  `deal` int(1) NOT NULL,   
  `deal_date` date default NULL,   
  `remark` text,   
  `last_push_time` datetime default NULL,   
  `push_times` int(11) default '1',   
  `first_set_ok_time` datetime default NULL,   
  `last_set_ok_time` datetime default NULL,   
  PRIMARY KEY  (`id`),   
  UNIQUE KEY `date` (`date`,`hash`)   
) ENGINE=InnoDB  DEFAULT CHARSET=utf8; 
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

UNIQUE KEY的用途：主要是用来防止数据插入的时候重复的。 

1，创建表时 

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE Persons   
(   
Id_P int NOT NULL,   
LastName varchar(255) NOT NULL,   
FirstName varchar(255),   
Address varchar(255),   
City varchar(255),   
UNIQUE (Id_P)   
);
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

 

如果需要命名 UNIQUE 约束，以及为多个列定义 UNIQUE 约束，请使用下面的 SQL 语法： 

![复制代码](https://common.cnblogs.com/images/copycode.gif)

```
CREATE TABLE Persons   
(   
Id_P int NOT NULL,   
LastName varchar(255) NOT NULL,   
FirstName varchar(255),   
Address varchar(255),   
City varchar(255),   
CONSTRAINT uc_PersonID UNIQUE (Id_P,LastName)   
);
```

![复制代码](https://common.cnblogs.com/images/copycode.gif)

 

2，当表已被创建时，如需在 "Id_P" 列创建 UNIQUE 约束，请使用下列 SQL： 

```
ALTER TABLE Persons   
ADD UNIQUE (Id_P);
```

 

 

如需命名 UNIQUE 约束，并定义多个列的 UNIQUE 约束，请使用下面的 SQL 语法： 

```
ALTER TABLE Persons   
ADD CONSTRAINT uc_PersonID UNIQUE (Id_P,LastName);  
```

 

3，撤销 UNIQUE 约束 

如需撤销 UNIQUE 约束，请使用下面的 SQL： 

```
ALTER TABLE Persons   
DROP INDEX uc_PersonID;  
```



## 转自
[mysql 中 key 、primary key 、unique key 与 index 区别 - 雪山上的蒲公英 - 博客园](https://www.cnblogs.com/zjfjava/)