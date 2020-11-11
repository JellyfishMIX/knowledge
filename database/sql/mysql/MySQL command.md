# MySQL command



## 数据库

### 查询所有数据库

```mysql
show databases;
```

### 切换选中的数据库

假设有一个数据库叫 o2o

```mysql
use o2o;
```

### 查询数据库中所有表

需要先选中某个数据库

```mysql
show tables;
```

### 查询某张完整的表

假设某张表叫 tb_area

```mysql
select * from tb_area;
```

