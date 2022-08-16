# Java数据类型和MySql数据类型对应表（转）



| **类型名称** | **显示长度** | **数据库类型**        | **JAVA类型**         | **JDBC类型索引(int)** | **描述**           |
| ------------ | ------------ | --------------------- | -------------------- | --------------------- | ------------------ |
|              |              |                       |                      |                       |                    |
| VARCHAR      | L+N          | VARCHAR               | java.lang.String     | 12                    |                    |
| CHAR         | N            | CHAR                  | java.lang.String     | 1                     |                    |
| BLOB         | L+N          | BLOB                  | java.lang.byte[]     | -4                    |                    |
| TEXT         | 65535        | VARCHAR               | java.lang.String     | -1                    |                    |
|              |              |                       |                      |                       |                    |
| INTEGER      | 4            | INTEGER UNSIGNED      | java.lang.Long       | 4                     |                    |
| TINYINT      | 3            | TINYINT UNSIGNED      | java.lang.Integer    | -6                    | JDBC TYPE: TINYINT |
| SMALLINT     | 5            | SMALLINT UNSIGNED     | java.lang.Integer    | 5                     |                    |
| MEDIUMINT    | 8            | MEDIUMINT UNSIGNED    | java.lang.Integer    | 4                     |                    |
| BIT          | 1            | BIT                   | java.lang.Boolean    | -7                    |                    |
| BIGINT       | 20           | BIGINT UNSIGNED       | java.math.BigInteger | -5                    |                    |
| FLOAT        | 4+8          | FLOAT                 | java.lang.Float      | 7                     |                    |
| DOUBLE       | 22           | DOUBLE                | java.lang.Double     | 8                     |                    |
| DECIMAL      | 11           | DECIMAL               | java.math.BigDecimal | 3                     |                    |
| BOOLEAN      | 1            | 同TINYINT             |                      |                       |                    |
|              |              |                       |                      |                       |                    |
| ID           | 11           | PK (INTEGER UNSIGNED) | java.lang.Long       | 4                     |                    |
|              |              |                       |                      |                       |                    |
| DATE         | 10           | DATE                  | java.sql.Date        | 91                    |                    |
| TIME         | 8            | TIME                  | java.sql.Time        | 92                    |                    |
| DATETIME     | 19           | DATETIME              | java.sql.Timestamp   | 93                    |                    |
| TIMESTAMP    | 19           | TIMESTAMP             | java.sql.Timestamp   | 93                    |                    |
| YEAR         | 4            | YEAR                  | java.sql.Date        | 91                    |                    |

对于bolb，一般用于对图片的数据库存储，原理是把图片打成二进制，然后进行的一种存储方式，在java中对应byte［］数组。

对于boolen类型，在mysql数据库中，个人认为用int类型代替较好，对bit操作不是很方便，尤其是在具有web页面开发的项目中，表示0/1，对应java类型的Integer较好。



MYSQL数据库类型与JAVA类型对应表

| 类型名称  | 显示长度 | 数据库类型            | JAVA类型             | JDBC类型索引(int) |
| --------- | -------- | --------------------- | -------------------- | ----------------- |
| VARCHAR   | L+N      | VARCHAR               | java.lang.String     | 12                |
| CHAR      | N        | CHAR                  | java.lang.String     | 1                 |
| BLOB      | L+N      | BLOB                  | java.lang.byte[]     | -4                |
| TEXT      | 65535    | VARCHAR               | java.lang.String     | -1                |
| INTEGER   | 4        | INTEGER UNSIGNED      | java.lang.Long       | 4                 |
| TINYINT   | 3        | TINYINT UNSIGNED      | java.lang.Integer    | -6                |
| SMALLINT  | 5        | SMALLINT UNSIGNED     | java.lang.Integer    | 5                 |
| MEDIUMINT | 8        | MEDIUMINT UNSIGNED    | java.lang.Integer    | 4                 |
| BIT       | 1        | BIT                   | java.lang.Boolean    | -7                |
| BIGINT    | 20       | BIGINT UNSIGNED       | java.math.BigInteger | -5                |
| FLOAT     | 4+8      | FLOAT                 | java.lang.Float      | 7                 |
| DOUBLE    | 22       | DOUBLE                | java.lang.Double     | 8                 |
| DECIMAL   | 11       | DECIMAL               | java.math.BigDecimal | 3                 |
| BOOLEAN   | 1        | 同TINYINT             |                      |                   |
| ID        | 11       | PK (INTEGER UNSIGNED) | java.lang.Long       | 4                 |
| DATE      | 10       | DATE                  | java.sql.Date        | 91                |
| TIME      | 8        | TIME                  | java.sql.Time        | 92                |
| DATETIME  | 19       | DATETIME              | java.sql.Timestamp   | 93                |
| TIMESTAMP | 19       | TIMESTAMP             | java.sql.Timestamp   | 93                |
| YEAR      | 4        | YEAR                  | java.sql.Date        | 91                |



## 转自

[Java数据类型和MySql数据类型对应表 - 草原和大树 - 博客园](https://www.cnblogs.com/jembai/archive/2009/08/20/1550683.html)

[练习两年半的攻城狮 - 博客园](https://www.cnblogs.com/jiezai/)