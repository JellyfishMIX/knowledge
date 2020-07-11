# MyBatis



## 预编译

### 定义
SQL 预编译指的是数据库驱动在发送 SQL 语句和参数给 DBMS 之前对 SQL 语句进行编译，这样 DBMS 执行 SQL 时，就不需要重新编译。

### 为什么需要预编译
- JDBC 中使用对象 PreparedStatement 来抽象预编译语句，使用预编译。预编译阶段可以优化 SQL 的执行。预编译之后的 SQL 多数情况下可以直接执行，DBMS 不需要再次编译，越复杂的SQL，编译的复杂度将越大，预编译阶段可以合并多次操作为一个操作。同时预编译语句对象可以重复利用。把一个 SQL 预编译后产生的 PreparedStatement 对象缓存下来，下次对于同一个SQL，可以直接使用这个缓存的 PreparedState 对象。Mybatis默认情况下，将对所有的 SQL 进行预编译。
- 可以防止sql注入。



## 防止sql注入

MyBatis框架作为一款半自动化的持久层框架，其SQL语句都要我们自己手动编写，这个时候当然需要防止SQL注入。其实，MyBatis的SQL是一个具有“输入+输出”的功能，类似于函数的结构，如下：

```xml
<select id= "orderblog" resulttype= "Blog" parametertype= "map" >

SELECT id,title,author,content

From blog

ORDER by #{orderparam}

</select>
```

这里，`parametertype` 表示输入的参数类型，`resulttype` 表示了输出的参数类型。回应上文，如果我们想防止SQL注入，理所当然地要在输入参数上下功夫。上面代码中黄色高亮即输入参数在SQL中拼接的部分，传入参数后，打印出执行的SQL语句，会看到SQL是这样的：

```sql
SELECT id,title,author,content FROM blog WHERE id = ?
```

不管输入什么参数，打印出的SQL是这样的。这是因为MyBatis启用了预编译功能，在SQL执行前，会先将上面的SQL在数据库驱动程序进行预编译。执行时，直接使用编译好的变量，替换占位符`?`就可以了。因为SQL注入只能对sql语句编译过程起作用，所以这样的方式就很好地避免了SQL注入的问题。

### 为什么sql注入只能对编译过程起作用？

sql注入的原理是把一段sql语句当作参数传入，想让这段被传入的sql语句执行，必须要在编译阶段编译成可执行语句。而使用`#{}`占位符方式，会把参数转为对应的数据类型。而某个数据类型（即某个变量）是不能在数据库中作为sql编译后的可执行语句操作数据库的，（因为此变量是按照对应数据类型被编译，而不是按照sql语句被编译）。



## #{}和${}的区别

Mybatis本身是基于JDBC封装的。

`#{param}` 是预编译处理（PreparedStatement）范畴的。

`${param}` 是字符串替换。

Mybatis在处理 `#` 时，会调用PreparedStatement的set系列方法来赋值，处理 `${param}` 替换成变量的值。

- 对于mapper

  `#{param}` 会产生PreparedStatement的占位符。

  `${param}` 直接替换花括号里的 `param`，变为 `param的值`。

- 对于配置文件
  `${key}` 用来替换环境属性中key指定的value。

详细请参考[mybatis – MyBatis 3](https://mybatis.org/mybatis-3/sqlmap-xml.html#String_Substitution)



## generator的SQL注入

为了提高开发效率，一些generator工具被开发出来，generator是一个从数据库结构 自动生成实体类、Mapper接口以及对应的XML文件的工具。常见的generator有mybatis-generator，renren-generator等。

mybatis-generator是mybatis官方的一款generator。在mybatis-generator自动生成的SQL语句中，order by使用的是 `$`，也就是简单的字符串拼接，这种情况下极易产生SQL注入。需要开发者特别注意。

不过，mybatis-generator产生的like语句和in语句全部都是使用 `#`，都是非常安全的实现。



## 引用/参考

[MyBatis面试题 - ThinkWon - CSDN](https://blog.csdn.net/ThinkWon/article/details/101292950)

[MyBatis如何防止SQL注入](https://blog.csdn.net/fmwind/article/details/59110918)

[mybatis中的#和$的区别 ？ - 叶锦昌的回答 - 知乎 ](https://www.zhihu.com/question/26914370/answer/51095713)

[mybatis中的#和$的区别 ？ - 侯云飞的回答 - 知乎](https://www.zhihu.com/question/26914370/answer/150328023)