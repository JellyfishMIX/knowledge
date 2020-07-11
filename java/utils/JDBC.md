# JDBC



## JDBC六大步骤

1. 注册驱动

2. 获取连接

3. 创建语句对象

4. 执行sql

5. 处理语句集

6. 关闭连接

```
// 加载jdbc驱动 
Class.forName("com.mysql.jdbc.Driver");

// 建立连接 
con=DriverManager.getConnection(url,user,password);

// 创建语句执行者（stateMent用于执行不带参数的简单sql语句，PreparedStatement用于执行带参数的预编译sql语句能够预防sql注入，CallableStatement提供了一种标准形式的调用存储过程的方法）
stmt=con.createStatement();  

// 执行sql
stmt.execute(“sql语句”); 

// 处理语句集
rs=stmt.executeQuery("sql查询语句");

// 关闭连接
close();
```



## ResultSet

ResultSet表示数据库结果集的数据表，通常通过执行查询数据库的语句生成。

ResultSet 接口提供用于从当前行获取列值的获取 方法（getBoolean、getLong 等）。可以使用列的索引编号或列的名称获取值。一般情况下，使用列索引较为高效。**列从 1 开始编号**。为了获得最大的可移植性，应该按从左到右的顺序读取每行中的结果集列，每列只能读取一次。

### 特点

- 具有指向其当前数据行的光标。最初，光标被置于第一行之前，next()方法将光标移动到下一行。因为该方法在ResultSet对象没有下一行时返回false，所以可以在while循环中使用它来迭代结果集。

- 提供用于从当前行获取列值的获取方法（`getInt()`, `getString()`等），可以使用列的索引编号或列的名称获取值。其中，列从1开始编号。（建议使用列名称获取列值） 

- 用作获取方法的输入的列名称不区分大小写。