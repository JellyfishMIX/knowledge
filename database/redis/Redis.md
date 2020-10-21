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



## redis 的过期策略以及内存淘汰机制

**(1）redis 的过期策略**

​	A、定期删除策略。用一个定时器来负责检查 key，过期则删除 key，注意这里并不是检查所有的 key 而是随机抽取进行检查。定期策略虽然让内存及时释放，但也会额外消耗 CPU 资源，通常 CPU 应该将时间尽量用于处理业务请求，而不是删除 key。

​	B、惰性删除策略。在你获取某个 key 的时候，redis 会检查一下，这个 key 如果设置了过期时间那么是否过期了，如果过期则删除该 key。

**(2) 内存淘汰机制**

如果定期删除没删除 key，然后也没及时去请求 key，即惰性删除也没生效，持续下去 redis 的内存会越来越高，当超过 redis 设置的内存最大使用量时，就会进行内存数据淘汰。redis 有 6 种淘汰策略：

| 策略            | 描                                                       |
| :-------------- | :------------------------------------------------------- |
| volatile-lru    | 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰     |
| volatile-ttl    | 从已设置过期时间的数据集中挑选将要过期的数据淘汰         |
| volatile-random | 从已设置过期时间的数据集中任意选择数据淘汰               |
| allkeys-lru     | 从所有数据集中挑选最近最少使用的数据淘汰                 |
| allkeys-random  | 从所有数据集中任意选择数据进行淘汰                       |
| noeviction      | 当内存不足以容纳新写入数据时，新写入操作会报错。很少使用 |

**点评：**

​	注意这里的 6 种机制，前缀 volatile 和 allkeys 用于区分淘汰数据的数据集是从**已设置过期时间的数据集还是从全部数据集**中选取，后面的 lru、ttl 以及 random 是三种不同的淘汰策略，再加上一种 no-enviction 永不回收的策略。其中最常使用的是 volatile-lru/allkeys-lru。



## redis 的持久化方式

redis 提供两种持久化方式。一种是 **RDB（Redis DataBase）**，用数据集快照的方式，定时将 redis 存储的数据生成快照并存储到磁盘等介质上；另外一种是 **AOF（Append -only file）**，是指所有的命令行记录以 redis 命令请求协议的格式完全持久化存储) 保存为 aof 文件。

**追问 1：RDB 和 AOF 各自的优缺点是什么？**

**(1) RDB 的优点：**

​	A 特别适合备份；

​	B. 性能最大化，fork 子进程来完成写操作，让主进程继续处理命令且不会进行任何 IO 操作的，这样就确保了 redis 极高的性能；

​	C. 相对于数据集大时，比 AOF 的启动效率更高。
**(2) RDB 的缺点：**
数据安全性低。RDB 是间隔一段时间进行持久化，如果持久化之间 redis 发	生故障，会发生数据丢失。所以这种方式更适合数据要求不严谨的时候；

 **(3) AOF 的优点：**
​	A. 数据安全，aof 持久化可以配置 append fsync 属性，比如无 fsync，每秒钟一次 fsync，或者每次执行写入命令时 fsync，一般只会丢失一秒钟的数据，或者最后一次执行的数据，对缓存来说，这已经足够。
​	B. 某些场景下还可以恢复数据。比如说某同学在操作 redis 时，不小心执行了 FLUSHALL，导致 redis 内存中的数据全部被清空了。如果 AOF 文件还没有被重写（rewrite），我们就可以用最快的速度暂停 redis 并编辑 AOF 文件，将最后一行的 FLUSHALL 命令删除，然后重启 redis，就可以恢复 redis 的所有数据到 FLUSHALL 之前的状态了。

​		**(4) AOF 的缺点：**

​	A. AOF 文件比 RDB 文件大，且根据不同的 fsync 策略，其恢复速度可能较慢；

```
	B 数据集大的时候，比 rdb 启动效率低。
```

**RDB 和 AOF 对比**：.

| 命令       | RDB    | AOF          |
| :--------- | :----- | :----------- |
| 启动优先级 | 低     | 高           |
| 体积       | 小     | 大           |
| 恢复速度   | 快     | 慢           |
| 数据安全性 | 丢数据 | 根据策略决定 |
| 轻重       | 重     | 轻           |

**追问 2： AOF 文件太大会怎么样？**

AOF 文件过大时，后台会自动地对 AOF 进行重写（rewrite)，重写时会压缩 AOF 文件的内容，只保留可以恢复数据的最小指令集。比如说，假如我们调用了 100 次 INCR 指令，在 AOF 文件中就要存储 100 条指令，但这明显是很低效的，完全可以把这 100 条指令合并成一条 SET 指令。

在进行 AOF 重写时，仍然是采用先写临时文件，全部完成后再替换的流程，所以断电、磁盘满等问题都不会影响 AOF 文件的可用性。



## 引用/参考

[Redis查询当前库有多少个 key - joshua317 - 博客园](https://www.cnblogs.com/joshua317/p/5825127.html)

[Spring-Data-Redis使用总结 - 陈俊龙 - OSCHINA](https://my.oschina.net/u/1048116/blog/688719)

[高薪之路--Java面试题精选集 - jiehao - 慕课网](https://www.imooc.com/read/67)