# redis -rdm 原生命令对照表

命令执行效果可参考rdm(redis desktop manager)。



## key

### delete

```
DEL <key>
```

e.g.

```
DEL "testString"
```

### expire

```
EXPIRE <key> <timeout>
```

timeout单位为秒

e.g.

```
EXPIRE "testList1" 10
```



## string

### stringNew

```
SET <key> <value>
```

e.g.

```
SET "newString" "newValue"
```

### stringSave

```
SET <key> <value>
```

e.g.

```
SET "testString" "testValue99"
```



## list

### listNew

```
LLEN <key>
LPUSH <key> <value>
```

e.g.

```
LLEN "testListNew"
LPUSH "testListNew" "888"
```

### listSave

```
LSET <key> <rowIndex> <value>
```

e.g.

```
LSET "testList" 3 "99"
```

### listAddRow

```
LPUSH <key> <value>
```

e.g.

```
LPUSH "testList" "testValue123"
```

### listDeleteRow

```
LSET <key> <rowIndex> <固定字符串："---VALUE_REMOVED_BY_HOPSONONE-PROCESS-SERVICE---">
LREM <key> <count> <固定字符串："---VALUE_REMOVED_BY_HOPSONONE-PROCESS-SERVICE---">
```

- rowIndex: List中的元素索引号。
- 根据参数 count 的值，移除列表中与参数 value 相等的元素。
  - count > 0 : 从表头开始向表尾搜索，移除与 value 相等的元素，数量为 count。
  - count < 0 : 从表尾开始向表头搜索，移除与 value 相等的元素，数量为 count 的绝对值。
  - count = 0 : 移除表中所有与 value 相等的值。

e.g.

```
LSET "testList" 4 "---VALUE_REMOVED_BY_HOPSONONE-PROCESS-SERVICE---"
LREM "testList" 0 "---VALUE_REMOVED_BY_HOPSONONE-PROCESS-SERVICE---"
```

- LREM中，0表示移除List中所有与 `value` 相等的值。



## hash

### hashNew

```
HLEN <key>
HSETNX <key> <hashKey> <hashValue>
```

e.g.

```
HLEN "testHashNew"
HSETNX "testHashNew" "testKey" "testValue"
```

### hashSave

```
HDEL <key> <originalHashKey>
HSET <key> <newHashKey> <hashValue>
```

如果只修改hashValue，originalHashKey和newHashKey可以相同。

e.g.

只修改hashValue

```
HDEL "testHash" "xiaoWang"
HSET "testHash" "xiaoWang" "23"
```

同时修改hashKey和hashValue

```
HDEL "testHash" "xiaoWang"
HSET "testHash" "xiaoWang1" "24"
```

### hashAddRow

```
HSETNX <key> <hashKey> <hashValue>
```

e.g.

```
HSETNX "testHash" "daWang" "33"
```

### hashDeleteRow

```
HDEL <key> <hashKey>
```

e.g.

```
HDEL "testHash" "xiaowang"
```



## set

### setNew

```
SCARD <key>
SADD <key> <value>
```

- SCARD命令 返回集合 `key` 的基数(集合中元素的数量)。
- SADD命令 将一个或多个成员元素加入到集合中，已经存在于集合的成员元素将被忽略。
  - 假如集合 key 不存在，则创建一个只包含添加的元素作成员的集合。
  - 当集合 key 不是集合类型时，返回一个错误。

e.g.

```
SCARD "testSetNew"
SADD "testSetNew" "666"
```

### setSave

```
SREM <key> <originalValue>
SADD <key> <newValue>
```

e.g.

```
SREM "testSet" "testValue"
SADD "testSet" "testValue1"
```

### setAddRow

```
SADD <key> <value>
```

e.g.

```
SADD "testSet" "666"
```

### setDeleteRow

```
SREM <key> <value>
```

e.g.

```
SREM "testSet" "111"
```



## zset

### zsetNew

```
ZCARD <key>
ZADD <key> <score> <value>
```

e.g.

```
ZCARD "testZSet"
ZADD "testZSet" 1 "testValue"
```

### zsetSave

```
ZREM <key> <originalValue>
ZADD <key> <score> <newValue>
```

如果只修改score，originalValue和newValue可以相同。

e.g.

- 只修改score

  ```
  ZREM "testZSet" "testValue"
  ZADD "testZSet" 10 "testValue"
  ```

- 同时修改score和value

  ```
  ZREM "testZSet" "testValue"
  ZADD "testZSet" 10 "testValue1"
  ```

### zsetAddRow

```
ZADD <key> <score> <value>
```

e.g.

```
ZADD "testZSet" 99 "testValue1"
```

### zsetDeleteRow

```
ZREM <key> <value>
```

e.g.

```
ZREM "testZSet" "testValue1"
```

