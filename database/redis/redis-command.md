# redis-command



## HASH

1. 命令参数：
   HSET key field value
   HSETNX key field value

2. 作用区别：
   HSET 将哈希表 key 中的域 field 的值设为 value 。如果 key 不存在，一个新的哈希表被创建并进行 HSET 操作。如果域 field 已经存在于哈希表中，旧值将被覆盖。

   HSETNX 将哈希表 key 中的域 field 的值设置为 value ，当且仅当域 field 不存在。若域 field 已经存在，该操作无效。

   如果 key 不存在，一个新哈希表被创建并执行 HSETNX 命令。

3. 返回值区别：
   HSET：如果 field 是哈希表中的一个新建域，并且值设置成功，返回 1 。如果哈希表中域 field 已经存在且旧值已被新值覆盖，返回 0 。
   HSETNX：设置成功，返回 1 。如果给定域已经存在且没有操作被执行，返回 0 。