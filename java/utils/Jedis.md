# Jedis



## Jedis set

四个方法的定义如下：

1、String    set(String key, String value)
2、String    set(String key, String value, String nxxx) 
3、String    set(String key, String value, String nxxx, String expx, int time) 
4、String    set(String key, String value, String nxxx, String expx, long time)
功能都是一样的，“Set the string value as value of the key.” 将string类型的value 放到key的value上，返回值都是 String。

1、把key、value set到redis中，隐含覆盖，默认的ttl是-1（永不过期）

2、根据第三个参数，把key、value set到redis中

    nx ： not exists, 只有key 不存在时才把key value set 到redis
    
    xx ： is exists ，只有 key 存在是，才把key value set 到redis

 


3、第四个参数expx参数有两个值可选 ：

          ex ： seconds 秒
    
          px :   milliseconds 毫秒
    
     使用其他值，抛出 异常 ： redis.clients.jedis.exceptions.JedisDataException : ERR syntax error 

4、第五个参数虽然有int和long，但实际最后都会转换成String类型，所以都无所谓。

最后，返回值String，如果写入成功是“OK”，写入失败返回空（在nxxx的时候，也是）



## 引用/参考

[Jedis set 的四个重载方法详解 - Love_云宝儿 - CSDN](https://blog.csdn.net/m0_38084879/article/details/103903789)

