# Redis



## 关键词

- 命中：从cache中获取数据，取到后返回。
- 失效：cache中的生存时间到了。
- 更新：把数据存到关系型数据库中，再更新到cache中。



## Redis查询当前库有多少个 key

- `info` 可以看到所有库的key数量。

- `dbsize` 则是当前库key的数量。

- `keys *` 这种数据量小还可以，大的时候可以直接搞死生产环境。

- `dbsize` 和 `keys *` 统计的key数可能是不一样的，如果没记错的话，`keys *` 统计的是当前db有效的key，而 `dbsize` 统计的是所有未被销毁的key（有效和未被销毁是不一样的，具体可以了解redis的过期策略）。



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



## Redis核心配置文件

redis.conf（windows版本下叫redis.windows.conf）

### 运行时配置更改

- Redis允许在运行的过程中，在不重启服务器的情况下更改服务器配置，同时也支持 使用特殊的[CONFIG SET](http://www.redis.cn/commands/config-set.html)和 [CONFIG GET](http://www.redis.cn/commands/config-get.html)命令用编程方式查询并设置配置。

- 并非所有的配置指令都支持这种使用方式，但是大部分是支持的。更多相关的信息请查阅[CONFIG SET](http://www.redis.cn/commands/config-set.html)和 [CONFIG GET](http://www.redis.cn/commands/config-get.html)页面。

- 需要确保的是在通过[CONFIG SET](http://www.redis.cn/commands/config-set.html)命令进行的设置的同时，也需在 redis.conf文件中进行了相应的更改。 未来Redis有计划提供一个[CONFIG REWRITE](http://www.redis.cn/commands/config-rewrite.html)命令在不更改现有配置文件的同时， 根据当下的服务器配置对redis.conf文件进行重写。



## 引用/参考

[Redis查询当前库有多少个 key - joshua317 - 博客园](https://www.cnblogs.com/joshua317/p/5825127.html)

[Spring-Data-Redis使用总结 - 陈俊龙 - OSCHINA](https://my.oschina.net/u/1048116/blog/688719)