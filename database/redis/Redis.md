# Redis



## 关键词

- 命中：从cache中获取数据，取到后返回。
- 失效：cache中的生存时间到了。
- 更新：把数据存到关系型数据库中，再更新到cache中。



## Redis查询当前库有多少个 key

- info可以看到所有库的key数量。

- dbsize则是当前库key的数量。

- keys *这种数据量小还可以，大的时候可以直接搞死生产环境。

- dbsize和keys *统计的key数可能是不一样的，如果没记错的话，keys *统计的是当前db有效的key，而dbsize统计的是所有未被销毁的key（有效和未被销毁是不一样的，具体可以了解redis的过期策略）。



## 列出所有数据库

Redis数据库的数量是固定的，并在配置文件中设置。默认情况下，你有16个数据库。每个数据库都由一个数字（而不是名称）来标识。

你可以使用以下命令来了解数据库的数量：

```javascript
CONFIG GET databases
1) "databases"
2) "16"
```

也可以使用以下命令列出定义了某些键的数据库：

```javascript
INFO keyspace
# Keyspace
db0:keys=10,expires=0
db1:keys=1,expires=0
db3:keys=1,expires=0
```

请注意，你应该使用“redis-cli”客户端来运行这些命令，而不是telnet。如果你想使用telnet，那么你需要运行这些使用Redis协议格式化的命令。

例如：

```javascript
*2
$4
INFO
$8
keyspace

$79
# Keyspace
db0:keys=10,expires=0
db1:keys=1,expires=0
db3:keys=1,expires=0
```

你可以在这里找到Redis协议的描述：[Redis Protocol specification](https://redis.io/topics/protocol)



## 引用/参考

[Redis查询当前库有多少个 key - joshua317 - 博客园](https://www.cnblogs.com/joshua317/p/5825127.html)