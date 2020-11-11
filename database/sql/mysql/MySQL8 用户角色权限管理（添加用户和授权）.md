# MySQL 8 用户权限管理（添加用户和授权角色）



## 前言

mysql8有新的安全要求，不能像之前的版本那样一次性创建用户并授权。需要先创建用户，再进行授权操作。



## 添加用户和授权

### 创建用户

```mysql
create user 'username'@'host' identified by 'password';
```

username 为自定义的用户名。

host 为登录域名，host 为’%'时表示为 任意IP，为 localhost 时表示本机，或者填写指定的IP地址。

paasword 为密码。密码需要满足：至少1位大写字符，至少1位小写字符，至少1位特殊字符，至少1位数字。

### 为用户授权角色

```mysql
grant all privileges on *.* to 'username'@'%' with grant option;
```

`*.*` 第一个 `*` 表示所有数据库，第二个 `*` 表示所有数据表。如果不想授权全部那就把对应的 `*` 写成相应数据库或者数据表。

username 为指定的用户。

% 为该用户登录的域名。

### 授权后刷新权限

```mysql
flush privileges;
```



## 角色权限管理

### 查看所有用户

```mysql
select User, Host from mysql.user;
```

MySQL 数据库中通常都会出现多个拥有相同权限集合的用户，在之前版本中只有分别向多个用户授予和撤销权限才能实现单独更改每个用户的权限。在用户数量比较多的时候，这样的操作是非常耗时的。

MySQL 8.0 为了用户权限管理更容易，提供了一个角色管理的新功能。角色是指定的权限集合，和用户帐户一样可以对角色进行权限的授予和撤消。如果用户被授予角色权限，则该用户拥有该角色的权限。

MySQL 8.0 提供的角色管理功能如下：

```
CREATE ROLE 角色创建
DROP ROLE 角色删除
GRANT 为用户和角色分配权限
REVOKE 为用户和角色撤销权限
SHOW GRANTS 显示用户和角色的权限
SET DEFAULT ROLE 指定哪些帐户角色默认处于活动状态
SET ROLE 更改当前会话中的活动角色
CURRENT_ROLE() 显示当前会话中的活动角色复制代码
```

#### 创建角色并授予用户角色权限

这里我们以几种常见场景为例。

- 应用程序需要读/写权限。
- 运维人员需要完全访问数据库。
- 部分开发人员需要读取权限。
- 部分开发人员需要读写权限。

如果要向多个用户授予相同的权限集，则应按如下步骤来进行。

- 创建新的角色
- 授予角色权限
- 授予用户角色

首先，我们创建四个角色。为了清楚区分角色的权限，建议将角色名称命名得比较直观。

```mysql
mysql> CREATE ROLE 'app', 'ops', 'dev_read', 'dev_write';
```

> 注：角色名称格式类似于由用户和主机部分组成的用户帐户，如：role_name@host_name。如果省略主机部分，则默认为 “%”，表示任何主机。

创建好角色后，我们就给角色授予对应的权限。要授予角色权限，您可以使用 `GRANT`语句。

```mysql
# 以下语句是向 app 角色授予 wordpress 数据库的读写权限
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON wordpress.* TO 'app';
# 以下语句是向 ops 角色授予 wordpress 数据库的所有权限
mysql> GRANT ALL PRIVILEGES ON wordpress.* TO 'ops';
# 以下语句是向 dev_read 角色授予 wordpress 数据库的只读权限
mysql> GRANT SELECT ON wordpress.* TO 'dev_read';
# 以下语句是向 dev_write 角色授予 wordpress 数据库的写权限
mysql> GRANT INSERT, UPDATE, DELETE ON wordpress.* TO 'dev_write';
```

> 注：这里假定需授权的数据库名称为 wordpress。

最后根据实际情况，我们将指定用户加入到对应的角色。假设需要一个应用程序使用的帐户、一个运维人员帐户、一个是开发人员只读帐户和两个开发人员读写帐户。

- 创建新用户

```mysql
# 应用程序帐户
mysql> CREATE USER 'app01'@'%' IDENTIFIED BY '000000';
# 运维人员帐户
mysql> CREATE USER 'ops01'@'%' IDENTIFIED BY '000000';
# 开发人员只读帐户
mysql> CREATE USER 'dev01'@'%' IDENTIFIED BY '000000';
# 开发读写帐户
mysql> CREATE USER 'dev02'@'%' IDENTIFIED BY '000000';
mysql> CREATE USER 'dev03'@'%' IDENTIFIED BY '000000';
```

- 给用户分配角色

```mysql
mysql> GRANT app TO 'app01'@'%';
mysql> GRANT ops TO 'ops01'@'%';
mysql> GRANT dev_read TO 'dev01'@'%';复制代码
```

如果要将多个用户同时加入多个角色，可以使用类似语句。

```mysql
mysql> GRANT dev_read, dev_write TO 'dev02'@'%', 'dev03'@'%';
```

#### 检查角色权限

要验证角色是否正确分配，可以使用 `SHOW GRANTS` 语句。

```mysql
mysql> SHOW GRANTS FOR 'dev01'@'%';
+-------------------------------------+
| Grants for dev01@%                  |
+-------------------------------------+
| GRANT USAGE ON *.* TO `dev01`@`%`   |
| GRANT `dev_read`@`%` TO `dev01`@`%` |
+-------------------------------------+
2 rows in set (0.00 sec)
```

正如你所看到的，和之前版本不同的是 `SHOW GRANTS` 只返回授予角色。如果要显示角色所代表的权限，需要加上 `USING` 子句和授权角色的名称。

```mysql
mysql> SHOW GRANTS FOR 'dev01'@'%' USING dev_read;
+----------------------------------------------+
| Grants for dev01@%                           |
+----------------------------------------------+
| GRANT USAGE ON *.* TO `dev01`@`%`            |
| GRANT SELECT ON `wordpress`.* TO `dev01`@`%` |
| GRANT `dev_read`@`%` TO `dev01`@`%`          |
+----------------------------------------------+
3 rows in set (0.00 sec)
```

#### 设置默认角色

现在，如果您使用 dev01 用户帐户连接到 MySQL，并尝试访问 wordpress 数据库会出现以下错误。

```mysql
$ mysql -u dev01 -p000000
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.11 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> use wordpress;
ERROR 1044 (42000): Access denied for user 'dev01'@'%' to database 'wordpress'
```

这是因为在向用户帐户授予角色后，当用户帐户连接到数据库服务器时，它并不会自动使角色变为活动状态。

```mysql
# 调用 CURRENT_ROLE() 函数查看当前角色。
mysql> SELECT current_role();
+----------------+
| current_role() |
+----------------+
| NONE           |
+----------------+
1 row in set (0.00 sec)
```

这里返回 NONE，就意味着当前没有启用任何角色。要在每次用户帐户连接到数据库服务器时指定哪些角色应该处于活动状态，需用使用 `SET DEFAULT ROLE` 语句来指定。

```mysql
# 以下语句将把 dev01 帐户分配的所有角色都设置为默认值。
mysql> SET DEFAULT ROLE ALL TO 'dev01'@'%';
```

再次使用 dev01 用户帐户连接到 MySQL 数据库服务器并调用 `CURRENT_ROLE()` 函数，您将看到 dev01 用户帐户的默认角色。

```mysql
$ mysql -u dev01 -p000000
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 11
Server version: 8.0.11 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>

# 查看 dev01 用户帐户的默认角色。
mysql> SELECT current_role();
+----------------+
| current_role() |
+----------------+
| `dev_read`@`%` |
+----------------+
1 row in set (0.00 sec)
```

最后通过将当前数据库切换到 wordpress 数据库，并执行 `SELECT` 语句和 `DELETE` 语句来测试 dev01 帐户的权限。

```mysql
mysql> use wordpress;
Database changed

mysql> select  count(*) from wp_terms;
+----------+
| count(*) |
+----------+
|      357 |
+----------+
1 row in set (0.00 sec)

mysql> DELETE from wp_terms;
ERROR 1142 (42000): DELETE command denied to user 'dev01'@'172.22.0.1' for table 'wp_terms'
```

如上面结果所示，当我们发出 `DELETE` 语句时，就收到一个错误。因为 dev01 用户帐户只有读取访问权限。

#### 设置活动角色

用户帐户可以通过指定哪个授权角色处于活动状态来修改当前用户在当前会话中的有效权限。

- 将活动角色设置为 NONE，表示没有活动角色。

```mysql
mysql> SET ROLE NONE;
```

- 将活动角色设置为所有授予的角色。

```mysql
mysql> SET ROLE ALL;
```

- 将活动角色设置为由 `SET DEFAULT ROLE` 语句设置的默认角色。

```mysql
mysql> SET ROLE DEFAULT;
```

- 同时设置多个活动的角色。

```mysql
mysql> SET ROLE granted_role_1, granted_role_2, ...
```

#### 撤消角色或角色权限

正如可以授权某个用户的角色一样，也可以从用户帐户中撤销这些角色。要从用户帐户中撤销角色需要使用 `REVOKE` 语句。

```mysql
mysql> REVOKE role FROM user;
```

`REVOKE` 也可以用于修改角色权限。这不仅影响角色本身权限，还影响任何授予该角色的用户权限。假设想临时让所有开发用户只读，可以使用 `REVOKE` 从 dev_write 角色中撤消修改权限。我们先来看下用户帐户 dev02 撤消前的权限。

```mysql
mysql> SHOW GRANTS FOR 'dev02'@'%' USING 'dev_read', 'dev_write';
+----------------------------------------------------------------------+
| Grants for dev02@%                                                   |
+----------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `dev02`@`%`                                    |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `wordpress`.* TO `dev02`@`%` |
| GRANT `dev_read`@`%`,`dev_write`@`%` TO `dev02`@`%`                  |
+----------------------------------------------------------------------+
3 rows in set (0.00 sec)
```

接下来从 dev_write 角色中撤消掉修改权限。

```mysql
mysql> REVOKE INSERT, UPDATE, DELETE ON wordpress.* FROM 'dev_write';
Query OK, 0 rows affected (0.03 sec)
```

最后我们在来看看 dev02 用户帐户当前权限。

```mysql
mysql> SHOW GRANTS FOR 'dev02'@'%' USING 'dev_read', 'dev_write';
+-----------------------------------------------------+
| Grants for dev02@%                                  |
+-----------------------------------------------------+
| GRANT USAGE ON *.* TO `dev02`@`%`                   |
| GRANT SELECT ON `wordpress`.* TO `dev02`@`%`        |
| GRANT `dev_read`@`%`,`dev_write`@`%` TO `dev02`@`%` |
+-----------------------------------------------------+
3 rows in set (0.00 sec)
```

从上面的结果可以看出，角色中撤销权限会影响到该角色中任何用户的权限。因此 dev02 现在已经没有表修改权限（INSERT，UPDATE，和 DELETE 权限已经去掉）。如果要恢复角色的修改权限，只需重新授予它们即可。

```mysql
# 授予 dev_write 角色修改权限。
mysql> GRANT INSERT, UPDATE, DELETE ON wordpress.* TO 'dev_write';

# 再次查看 dev02 用户权限，修改权限已经恢复。
mysql> SHOW GRANTS FOR 'dev02'@'%' USING 'dev_read', 'dev_write';
+----------------------------------------------------------------------+
| Grants for dev02@%                                                   |
+----------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `dev02`@`%`                                    |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `wordpress`.* TO `dev02`@`%` |
| GRANT `dev_read`@`%`,`dev_write`@`%` TO `dev02`@`%`                  |
+----------------------------------------------------------------------+
3 rows in set (0.00 sec)
```

#### 删除角色

要删除一个或多个角色，可以使用 `DROP ROLE` 语句。

```mysql
mysql> DROP ROLE 'role_name', 'role_name', ...;
```

如同 `REVOKE` 语句一样，删除角色会从授权它的每个帐户中撤消该角色。例如，要删除 dev_read，dev_write角色，可使用以下语句。

```mysql
mysql> DROP ROLE 'dev_read', 'dev_write';
```

#### 复制用户帐户权限到另一个用户

MySQL 8.0 将每一个用户帐户视为角色，因此可以将用户帐户授予另一个用户帐户。例如：将一开发人员帐号权限复制到另一开发人员帐号。

- 创建一个新的开发用户帐户

```mysql
mysql> CREATE USER 'dev04'@'%' IDENTIFIED BY '000000';
Query OK, 0 rows affected (0.04 sec)
```

- 将 dev02 用户帐户的权限复制到 dev04 用户帐户

```mysql
mysql> GRANT 'dev02'@'%' TO 'dev04'@'%';
Query OK, 0 rows affected (0.09 sec)
```

- 查看 dev04 用户帐户的权限

```mysql
mysql> SHOW GRANTS FOR 'dev04'@'%' USING 'dev02';
+----------------------------------------------------------------------+
| Grants for dev04@%                                                   |
+----------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `dev04`@`%`                                    |
| GRANT SELECT, INSERT, UPDATE, DELETE ON `wordpress`.* TO `dev04`@`%` |
| GRANT `dev02`@`%` TO `dev04`@`%`                                     |
+----------------------------------------------------------------------+
3 rows in set (0.00 sec)
```



## MySQL 权限列表

以下操作都是以root身份登陆进行grant授权，以root@localhost身份登陆执行各种命令。共29个权限。

| 权限                    | 说明                                                         | 举例                                                         |
| :---------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| usage                   | 连接（登陆）权限，建立一个用户，就会自动授予其usage权限（默认授予）。    该权限只能用于数据库登陆，不能执行任何操作；且usage权限不能被回收，也即REVOKE用户并不能删除用户。 | mysql>  grant usage on *.* to 'root′@'localhost' identified by '123'; |
| file                    | 拥有file权限才可以执行  select ..into outfile和load data infile…操作，但是不要把file, process,  super权限授予管理员以外的账号，这样存在严重的安全隐患。 | mysql>  grant file on *.* to root@localhost;    mysql> load data infile '/home/mysql/pet.txt' into table pet; |
| super                   | 这个权限允许用户终止任何查询；修改全局变量的SET语句；使用CHANGE  MASTER，PURGE MASTER LOGS。 | mysql>  grant super on *.* to root@localhost;    mysql> purge master logs before 'mysql-bin.000006′; |
| select                  | 必须有select的权限，才可以使用select  table                  | mysql>  grant select on pyt.* to 'root′@'localhost';    mysql> select * from shop; |
| insert                  | 必须有insert的权限，才可以使用insert  into ….. values….      | mysql>  grant insert on pyt.* to 'root′@'localhost';    mysql> insert into shop(name) values('aa'); |
| update                  | 必须有update的权限，才可以使用update  table                  | mysql>  update shop set price=3.5 where article=0001 and dealer='A'; |
| delete                  | 必须有delete的权限，才可以使用delete  from ….where….(删除表中的记录) | mysql>  grant delete on pyt.* to 'root′@'localhost';    mysql> delete from table where id=1; |
| alter                   | 必须有alter的权限，才可以使用alter  table                    | mysql>  alter table shop modify dealer char(15);             |
| alter routine           | 必须具有alter  routine的权限，才可以使用{alter \|drop} {procedure\|function} | mysql>grant  alter routine on pyt.* to 'root′@' localhost ‘;    mysql> drop procedure pro_shop;    Query OK, 0 rows affected (0.00 sec) |
| create                  | 必须有create的权限，才可以使用create  table                  | mysql>  grant create on pyt.* to 'root′@'localhost';         |
| drop                    | 必须有drop的权限，才可以删除库、表、索引、视图等             | mysql>  drop database db_name;     mysql> drop table tab_name;    mysql> drop view vi_name;     mysql> drop index in_name; |
| create routine          | 必须具有create  routine的权限，才可以使用{create \|alter\|drop} {procedure\|function} | mysql>  grant create routine on pyt.* to 'root′@'localhost';    当授予create routine时，自动授予EXECUTE, ALTER ROUTINE权限给它的创建者： |
| create temporary tables | (注意这里是tables，不是table)                                | 必须有create  temporary tables的权限，才可以使用create temporary tables.    mysql> grant create temporary tables on pyt.* to  'root′@'localhost';    [mysql@mydev ~]$ mysql -h localhost -u root -p pyt    mysql> create temporary table tt1(id int); |
| create view             | 必须有create  view的权限，才可以使用create view              | mysql>  grant create view on pyt.* to 'root′@'localhost';    mysql> create view v_shop as select price from shop; |
| create user             | 要使用CREATE  USER，必须拥有mysql数据库的全局CREATE USER权限，或拥有INSERT权限。 | mysql>  grant create user on *.* to 'root′@'localhost';    或：mysql> grant insert on *.* to root@localhost; |
| show database           | 通过show  database只能看到你拥有的某些权限的数据库，除非你拥有全局SHOW DATABASES权限。    对于root@localhost用户来说，没有对mysql数据库的权限，所以以此身份登陆查询时，无法看到mysql数据库： | mysql>  show databases;                                      |
| show view               | 必须拥有show  view权限，才能执行show create view             | mysql>  show create view name;                               |
| index                   | 必须拥有index权限，才能执行[create  \|drop] index            | mysql>  grant index on pyt.* to root@localhost;    mysql> create index ix_shop on shop(article);    mysql> drop index ix_shop on shop; |
| excute                  | 执行存在的Functions,Procedures                               | mysql>  call pro_shoroot(0001,@a)；                          |
| event                   | event的使用频率较低建议使用root用户进行创建和维护。    要使event起作用，MySQL的常量GLOBAL event_scheduler必须为on或者是1 | mysql>  show global variables like 'event_scheduler';        |
| lock tables             | 必须拥有lock  tables权限，才可以使用lock tables              | mysql>  grant lock tables on pyt.* to root@localhost;    mysql> lock tables a1 read;    mysql> unlock tables; |
| references              | 有了REFERENCES权限，用户就可以将其它表的一个字段作为某一个表的外键约束。 |                                                              |
| reload                  | 必须拥有reload权限，才可以执行flush  [tables \| logs \| privileges] | mysql>  grant reload on pyt.* to root@localhost;    ERROR 1221 (HY000): Incorrect usage of DB GRANT and GLOBAL PRIVILEGES    mysql> grant reload on *.* to 'root′@'localhost';    Query OK, 0 rows affected (0.00 sec)    mysql> flush tables; |
| replication client      | 拥有此权限可以查询master  server、slave server状态。         | mysql>  grant Replication client on *.* to root@localhost;    或：mysql> grant super on *.* to root@localhost;    mysql> show master status; |
| replication slave       | 拥有此权限可以查看从服务器，从主服务器读取二进制日志。       | mysql>  grant replication slave on *.* to root@localhost;    mysql> show slave hosts;    Empty set (0.00 sec)    mysql>show binlog events; |
| Shutdown                | 关闭mysql权限                                                | [mysql@mydev  ~]$ mysqladmin shutdown                        |
| grant option            | 拥有grant  option，就可以将自己拥有的权限授予其他用户（仅限于自己已经拥有的权限） | mysql>  grant Grant option on pyt.* to root@localhost;    mysql> grant select on pyt.* to p2@localhost; |
| process                 | 通过这个权限，用户可以执行SHOW  PROCESSLIST和KILL命令。默认情况下，每个用户都可以执行SHOW PROCESSLIST命令，但是只能查询本用户的进程。 | mysql>  show processlist;                                    |
| all privileges          | 所有权限。with  grant option 可以连带授权                    | mysql>  grant all privileges on pyt.* to root@localhost with grant option; |

另外，管理权限（如 super， process， file等）不能够指定某个数据库，on后面必须跟 `*.*`。



## 扩展阅读

[MySQL 8.0用户和角色管理 - 运维之美 - 掘金](https://juejin.im/post/6844903655200538638)



## 引用/参考

[mysql8.0数据库添加用户和授权 - 暴走的JJ - CSDN](https://blog.csdn.net/qq_23859799/article/details/85862821)

[MySQL 8.0用户和角色管理 - 运维之美 - 掘金](https://juejin.im/post/6844903655200538638)

[你真的熟悉MySQL权限吗？ - MySQL轻松学 - 腾讯云社区](https://cloud.tencent.com/developer/article/1056271)