# Redis Java Client



## Redis的三个java客户端

Jedis api 在线网址：http://tool.oschina.net/uploads/apidocs/redis/clients/jedis/Jedis.html

redisson 官网地址：https://redisson.org/

redisson git项目地址：https://github.com/redisson/redisson

lettuce 官网地址：https://lettuce.io/

lettuce github项目地址：https://github.com/lettuce-io/lettuce-core

首先，在spring boot2之后，对redis连接的支持，默认就采用了lettuce。这就一定程度说明了lettuce 和Jedis的优劣。

### 概念：

Jedis：是老牌的Redis的Java实现客户端，提供了比较全面的Redis命令的支持，

Redisson：实现了分布式和可扩展的Java数据结构。

Lettuce：高级Redis客户端，用于线程安全同步，异步和响应使用，支持集群，Sentinel，管道和编码器。

### 优点：

Jedis：比较全面的提供了Redis的操作特性。

Redisson：促使使用者对Redis的关注分离，提供很多分布式相关操作服务，例如，分布式锁，分布式集合，可通过Redis支持延迟队列。

Lettuce：基于Netty框架的事件驱动的通信层，其方法调用是异步的。Lettuce的API是线程安全的，所以可以操作单个Lettuce连接来完成各种操作。

### 可伸缩：

Jedis：使用阻塞的I/O，且其方法调用都是同步的，程序流需要等到sockets处理完I/O才能执行，不支持异步。Jedis客户端实例不是线程安全的，所以需要通过连接池来使用Jedis。

Redisson：基于Netty框架的事件驱动的通信层，其方法调用是异步的。Redisson的API是线程安全的，所以可以操作单个Redisson连接来完成各种操作。

Lettuce：基于Netty框架的事件驱动的通信层，其方法调用是异步的。Lettuce的API是线程安全的，所以可以操作单个Lettuce连接来完成各种操作。

lettuce能够支持redis4，需要java8及以上。
lettuce是基于netty实现的与redis进行同步和异步的通信。

### lettuce和jedis比较：

jedis使直接连接redis server,如果在多线程环境下是非线程安全的，这个时候只有使用连接池，为每个jedis实例增加物理连接。

lettuce的连接是基于Netty的，连接实例（StatefulRedisConnection）可以在多个线程间并发访问，StatefulRedisConnection是线程安全的，所以一个连接实例可以满足多线程环境下的并发访问，当然这也是可伸缩的设计，一个连接实例不够的情况也可以按需增加连接实例。

默认情况，Lettuce 对所有非阻塞和非事务型操作，共享同一个线程安全的本地连接。可配置LettucePool，为阻塞和事务操作，提供独占连接。

Redisson实现了分布式和可扩展的Java数据结构，和Jedis相比，功能较为简单，不支持字符串操作，不支持排序、事务、管道、分区等Redis特性。Redisson的宗旨是促进使用者对Redis的关注分离，从而让使用者能够将精力更集中地放在处理业务逻辑上。

### 对比列表

|                   | Jedis                                                        | Lettuce                                                      | Redisson                                                     |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 地址              | https://github.com/xetorthio/jedis                           | https://lettuce.io/                                          | https://github.com/redisson/redisson                         |
| what              | 小且健全的redis java客户端，支持redis的所有特性和命令，如事务、管道、发布订阅。 | 线程安全的redis客户端，封装同步、异步、交互API。不执行阻塞和事务操作时，多线程可共享连接。 | 线程安全的redis客户端，支持多场景，提供基于redis的某些分布式服务解决方案。 |
| 实现与使用        | 实现简单，使用简单                                           | 实现较复杂，使用简单                                         | 实现复杂，使用简单                                           |
| 网络              | 阻塞IO                                                       | Netty                                                        | Netty                                                        |
| redis命令特性支持 | redis命令与特性提供健全的支持                                | redis命令与特性支持                                          | redis命令和特性支持                                          |
| 抽象封装程度      | 没有做特别的抽象，特性使用是否正确依赖使用者                 | 同步、异步、交互场景封装                                     | 丰富的数据模型、分布式服务、特性的封装，第三方框架的扩展实现 |
| 操作的维度        | 指令维度的操作                                               | 指令维度的操作                                               | 使用对象、服务将redis指令和业务分离，对象维度的操作，使用更方便，没有使用指令灵活，支持redisClient执行指令；分布式服务特性缺乏管理平台 |
| redis连接         | 每次操作均需要从连接池中获取连接，线程间不可以共享连接，高并发时需要考虑连接过多对client和server的影响 | 共享连接，连接是long-lived和线程安全的，而且自动重连         | 共享连接                                                     |
| client分片        | 支持，提供实现                                               | 不支持，未提供实现                                           | 不支持，针对特殊数据模型提供数据分片                         |
| read slave        | 未找到使用slave执行read指令的API，但可以自行实现             | 支持slave执行read指令                                        | 支持slave执行read指令                                        |
| 排他+超时         | 支持                                                         | 支持                                                         | 需要使用封装的数据模型                                       |
| 社区维护          | 社区维护好，版本更新快                                       | 社区维护一般，版本更新较慢                                   | 分为企业版和开源版，维护好，版本更新快                       |
| 特性表            | Sorting<br/>Connection handling<br/>Commands operating on any kind of values<br/>Commands operating on string values<br/>Commands operating on hashes<br/>Commands operating on lists<br/>Commands operating on sets<br/>Commands operating on sorted sets<br/>Transactions<br/>Pipelining<br/>Publish/Subscribe<br/>Persistence control commands<br/>Remote server control commands<br/>Connection pooling<br/>Sharding (MD5, MurmurHash)<br/>Key-tags for sharding<br/>Sharding with pipelining<br/>Scripting with pipelining<br/>Redis Cluster<br/> | synchronous, asynchronous and reactive usage<br/>Redis Sentinel<br/>Redis Cluster<br/>SSL and Unix Domain Socket connections<br/>Streaming API<br/>CDI and Spring integration<br/>Codecs (for UTF8/bit/JSON etc. representation of your data)<br/>multiple Command Interfaces<br/> | 各种封装的对象、服务、其他框架的扩展实现                     |

### 确定选型

#### client连接数对redis性能影响的角度

| 连接数 | QPS  |
| ------ | ---- |
| 100    | 12w  |
| 3w     | 6w   |
| 6w     | 5w   |

所以若每秒使用redis次数达数万，不建议用Jedis，因为不仅降低redis性能，而且降低client所在服务器的性能。

可选择Lettuce，Redisson。

#### client分片的角度  

 如果不使用redis cluster，而是采用client端分片，则选择Jedis。

#### 隔离redis具体使用的角度

 Redisson，将redis指令与业务隔离，直接使用数据模型、特性封装、分布式服务，而不用关心使用哪些redis指令。

### 总结：

优先使用Lettuce，如果需要分布式锁，分布式集合等分布式的高级特性，添加Redisson结合使用，因为Redisson本身对字符串的操作支持很差。



## Spring-data-redis的使用

Spring Boot Data Redis 默认依赖 lettuce。

### Jedis与Spring-data-redis的区别和关系

- Jedis：Jedis是Redis的Java客户端，通过它可以对Redis进行操作，与之功能相似的还包括Lettuce等。
- Spring-data-redis：Spring-data-redis对Redis的操作依赖Jedis或Lettuce，实际上是对Jedis、Lettuce这些客户端的封装，提供一套与客户端无关的api供应用使用，从而你在从一个redis客户端切换为另一个客户端，不需要修改业务代码。

### Spring和Spring-data-redis整合

通过 `.xml` 或 `.properties` 配置文件将 `Spring-data-redis` 的**连接池**、**Redis模板**注入到Spring容器中。Redis模板有`2`个，分别是`RedisTemplate`、`StringRedisTemplate`，这两个模板的区别是采用的序列化策略不一样，前者采用的是Java**原生的序列化**后者采用的是**String序列化**。模板的好处是为Redis的交互提供了高级抽象，用户无需关注Redis的连接管理、序列化等问题，把更多注意力放在业务上。

 **redis.properties：**

```properties
#ip地址
redis.host.ip=192.168.174.129
#端口号
redis.port=6379
#如果有密码
redis.password=
#客户端超时时间单位是毫秒 默认是2000
redis.timeout=3000

#最大空闲数
redis.maxIdle=6
#连接池的最大数据库连接数。设为0表示无限制,如果是jedis 2.4以后用redis.maxTotal
#redis.maxActive=600maxIdle
#控制一个pool可分配多少个jedis实例,用来替换上面的redis.maxActive,如果是jedis 2.4以后用该属性
redis.maxTotal=20
#最大建立连接等待时间。如果超过此时间将接到异常。设为-1表示无限制。
redis.maxWaitMillis=3000
#连接的最小空闲时间 默认1800000毫秒(30分钟)
redis.minEvictableIdleTimeMillis=300000
#每次释放连接的最大数目,默认3
redis.numTestsPerEvictionRun=4
#逐出扫描的时间间隔(毫秒) 如果为负数,则不运行逐出线程, 默认-1
redis.timeBetweenEvictionRunsMillis=30000
```

**spring-redis.xml：**

```xml
<!--加载配置文件-->
<bean:property-placeholder location="classpath:redis.properties" ignore-unresolvable="true"/>

<!--redis连接池配置-->
<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
    <!--最大空闲数 -->
    <property name="maxIdle" value="${redis.maxIdle}" />
    <!--连接池的最大数据库连接数 -->
    <property name="maxTotal" value="${redis.maxTotal}" />
    <!--最大建立连接等待时间 -->
    <property name="maxWaitMillis" value="${redis.maxWaitMillis}" />
    <!--逐出连接的最小空闲时间 默认1800000毫秒(30分钟) -->
    <property name="minEvictableIdleTimeMillis" value="${redis.minEvictableIdleTimeMillis}" />
    <!--每次逐出检查时 逐出的最大数目 如果为负数就是 : 1/abs(n), 默认3 -->
    <property name="numTestsPerEvictionRun" value="${redis.numTestsPerEvictionRun}" />
    <!--逐出扫描的时间间隔(毫秒) 如果为负数,则不运行逐出线程, 默认-1 -->
    <property name="timeBetweenEvictionRunsMillis" value="${redis.timeBetweenEvictionRunsMillis}" />
</bean>

<!--redis连接工厂-->
<bean id="connectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
    <!--连接池配置-->
    <property name="poolConfig" ref="jedisPoolConfig"></property>
    <!--redis地址-->
    <property name="hostName" value="${redis.host.ip}"></property>
    <!--redis密码-->
    <!--<property name="password" value="${redis.password}"></property>-->
    <!--redis端口-->
    <property name="port" value="${redis.port}"></property>
    <!--超时时间 毫秒-->
    <property name="timeout" value="${redis.timeout}"></property>
</bean>

<!-- 注入redis操作模板为Bean  自动装配-->
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="connectionFactory" />
    <!-- 指定redis中key-value的序列化方式（此处省略） -->
</bean>

<!-- 注入redis操作模板为Bean  自动装配-->
<bean id="stringRedisTemplate" class="org.springframework.data.redis.core.StringRedisTemplate">
    <property name="connectionFactory" ref="connectionFactory" />
</bean>
复制代码
```

### Spring Data Redis 对 Spring Cache 的支持

spring cache 是一个spring缓存组件，spring3.1以后提供了Spring  cache缓存抽象，和其他spring组件一样，例如spring事务等，采用注解驱动的方式，利用Spring AOP（动态代理）结合XML Schema来实现，采用这用方式对代码的侵入性极小，是一种轻量级的缓存抽象。

SDR对SpringCache提供支持，将原有的基于JVM容器的缓存，采用基于Redis缓存中间件来完成，实现了分布式缓存。

```
<cache:annotation-driven />
<bean id="cacheManager" class="org.springframework.data.redis.cache.RedisCacheManager">
	<constructor-arg ref="redisTemplate" />
</bean>
@Cacheable(value = "userCache",condition="#mobile ne null and #mobile ne ''",unless = "#result eq '' or #result eq null")
@Transactional(propagation = Propagation.NOT_SUPPORTED, readOnly = true)
@Override
public User getUserByMobile(String mobile) {
	System.out.println("execute getUserByMobile start...");
	User user = userDao.getUserByMobile(mobile);
	System.out.println("execute getUserByMobile end...");
	return user;
}
```

spring cache支持 SpEL语法

SpEL语法详见官方介绍：http://docs.spring.io/spring/docs/current/spring-framework-reference/htmlsingle/#expressions

```
@Cacheable triggers cache population
@CacheEvict triggers cache eviction
@CachePut updates the cache without interfering with the method execution
@Caching regroups multiple cache operations to be applied on a method
@CacheConfig shares some common cache-related settings at class-level
```

### SpringBoot中以注解方式使用Redis

首先，需要存入redis的对象，需要实现Serializable接口，否则在存入时会报错。

#### @Cacheable

```java
@Cacheable(cacheNames = "product", key = "123", unless = "#result.getCode() != 0")
```

- 第一次访问key的时候会进入到方法中，方法返回的对象会存储到redis中。下一次再访问这个key的时候，就不会进入方法，会直接从redis中拿数据。

- key不填写，为方法的参数。

- unless：执行存入缓存操作，除非（）条件成立才不存入缓存。使用场景：方法返回的结果有可能是错误的，首先判断一下，正确的方法返回结果才存入缓存。

- 注解的属性值可以使用SpEL表达式。

  ```java
  @Cacheable(cacheNames = "product", key = "#sellerId", condition = "#sellerId.length() > 3")
  public ResultVO list(String sellerId) {
      
  }
  ```

  condition属性的值为true，会进行存入缓存操作；如果值为false，不会进行存入缓存操作。

- condition和unless的区别：

  - condition对传入值生效，unless对结果result生效。

#### @CachePut

```java
@CachePut(cacheNames = "product", key = "123")
```

- 每次访问key都会进入到方法中，并且每次都会把返回的结果存入redis。

- 更新方法返回的对象，一定要和返回方法返回的对象保持一致。
- key不填写，为方法的参数。

#### @CacheEvict

```java
@CacheEvict(cacheNames = "product", key = "123")
```

方法执行后，可以清除redis中key对应的缓存。

key不填写，为方法的参数。

#### @CacheConfig

```java
@CacheConfig(cacheNames = "productInfo")
public class ProductServiceImpl implements ProductService {
    
}
```

#### 组合使用

两种方案：

- 查询API方法加上@Cacheable，更新API方法加上@CachePut。这样每次更新后会更新key对应的缓存。
- 查询API方法加上@Cacheable，更新API方法加上@CacheEvict。这样每次更新后会清除key对应的缓存，下次再通过key请求数据的时候，会先从关系型数据库中查询数据，然后存入redis。

### spring-data-redis的方法

将`Spring`和`Spring-data-redis`整合完成之后，一般为了方便使用模板，我们会将模板进一步封装成自己的Dao工具类，这里我仅封装几个操作，如下 **RedisDaoImpl.java**

```java
@Repository
public class RedisDaoImpl implements RedisDao {
    @Autowired
    StringRedisTemplate stringRedisTemplate;

    public void setString(Object redisKey, Object redisValue) {
        ValueOperations valueOperations = stringRedisTemplate.opsForValue();
        valueOperations.set(redisKey, redisValue);

    }

    public Object getString(Object redisKey) {
        ValueOperations valueOperations = stringRedisTemplate.opsForValue();
        return valueOperations.get(redisKey);
    }

    /**
     * @description: 通过redisKey 批量(map)设置redisValue(hash)
     * @param: [redisKey, redisValue]
     * @return: void
     * @author: Xue 8
     * @date: 2019/2/14
     */
    public void setHashAll(Object redisKey, Map<String,Object> redisValue){
        HashOperations hashOperations = stringRedisTemplate.opsForHash();
        hashOperations.putAll(redisKey, redisValue);
    }

    /**
     * @description: 通过redisKey、hashKey、hashValue设置单个redisValue(hash)
     * @param: [redisKey, hashKey, hashValue]
     * @return: void
     * @author: Xue 8
     * @date: 2019/2/14
     */
    public void setHash(Object redisKey, Object hashKey, Object hashValue){
        HashOperations hashOperations = stringRedisTemplate.opsForHash();
        hashOperations.put(redisKey, hashKey, hashValue);
    }

    /**
     * @description: 通过redisValue、hashKey获取hashValue
     * @param: [redisKey, hashKey]
     * @return: java.lang.Object
     * @author: Xue 8
     * @date: 2019/2/14
     */
    public Object getHashValue(Object redisKey, Object hashKey){
        HashOperations hashOperations = stringRedisTemplate.opsForHash();
        return hashOperations.get(redisKey, hashKey);
    }

    /**
     * @description: 通过redisKey获取hash
     * @param: [redisKey]
     * @return: java.util.Map<java.lang.String,java.lang.Object>
     * @author: Xue 8
     * @date: 2019/2/14
     */
    public Map<String,Object> getHash(Object redisKey){
        HashOperations hashOperations = stringRedisTemplate.opsForHash();
        return hashOperations.entries(redisKey);
    }

    /**
     * @description: 通过redisKey设置redisValue(List)
     * @param: [redisKey, redisValue]
     * @return: void
     * @author: Xue 8
     * @date: 2019/2/14
     */
    public void setList(Object redisKey, List<Object> redisValue){
        ListOperations listOperations = stringRedisTemplate.opsForList();
        listOperations.leftPushAll(redisKey, redisValue);
    }

}
复制代码
```

之后在需要操作到Redis的地方，直接调用RedisDaoImpl即可。

### 使用RedisTemplate自由操作Redis

Spring Cache 给我们提供了操作Redis缓存的便捷方法，但是也有很多局限性。比如说我们想单独设置一个缓存值的有效期怎么办？我们并不想缓存方法的返回值，我们想缓存方法中产生的中间值怎么办？此时我们就需要用到RedisTemplate这个类了，接下来我们来讲下如何通过RedisTemplate来自由操作Redis中的缓存。

#### RedisService

定义Redis操作业务类，在Redis中有几种数据结构，比如普通结构（对象），Hash结构、Set结构、List结构，该接口中定义了大多数常用操作方法。

```
/**
 * redis操作Service
 * Created by macro on 2020/3/3.
 */
public interface RedisService {

    /**
     * 保存属性
     */
    void set(String key, Object value, long time);

    /**
     * 保存属性
     */
    void set(String key, Object value);

    /**
     * 获取属性
     */
    Object get(String key);

    /**
     * 删除属性
     */
    Boolean del(String key);

    /**
     * 批量删除属性
     */
    Long del(List<String> keys);

    /**
     * 设置过期时间
     */
    Boolean expire(String key, long time);

    /**
     * 获取过期时间
     */
    Long getExpire(String key);

    /**
     * 判断是否有该属性
     */
    Boolean hasKey(String key);

    /**
     * 按delta递增
     */
    Long incr(String key, long delta);

    /**
     * 按delta递减
     */
    Long decr(String key, long delta);

    /**
     * 获取Hash结构中的属性
     */
    Object hGet(String key, String hashKey);

    /**
     * 向Hash结构中放入一个属性
     */
    Boolean hSet(String key, String hashKey, Object value, long time);

    /**
     * 向Hash结构中放入一个属性
     */
    void hSet(String key, String hashKey, Object value);

    /**
     * 直接获取整个Hash结构
     */
    Map<Object, Object> hGetAll(String key);

    /**
     * 直接设置整个Hash结构
     */
    Boolean hSetAll(String key, Map<String, Object> map, long time);

    /**
     * 直接设置整个Hash结构
     */
    void hSetAll(String key, Map<String, Object> map);

    /**
     * 删除Hash结构中的属性
     */
    void hDel(String key, Object... hashKey);

    /**
     * 判断Hash结构中是否有该属性
     */
    Boolean hHasKey(String key, String hashKey);

    /**
     * Hash结构中属性递增
     */
    Long hIncr(String key, String hashKey, Long delta);

    /**
     * Hash结构中属性递减
     */
    Long hDecr(String key, String hashKey, Long delta);

    /**
     * 获取Set结构
     */
    Set<Object> sMembers(String key);

    /**
     * 向Set结构中添加属性
     */
    Long sAdd(String key, Object... values);

    /**
     * 向Set结构中添加属性
     */
    Long sAdd(String key, long time, Object... values);

    /**
     * 是否为Set中的属性
     */
    Boolean sIsMember(String key, Object value);

    /**
     * 获取Set结构的长度
     */
    Long sSize(String key);

    /**
     * 删除Set结构中的属性
     */
    Long sRemove(String key, Object... values);

    /**
     * 获取List结构中的属性
     */
    List<Object> lRange(String key, long start, long end);

    /**
     * 获取List结构的长度
     */
    Long lSize(String key);

    /**
     * 根据索引获取List中的属性
     */
    Object lIndex(String key, long index);

    /**
     * 向List结构中添加属性
     */
    Long lPush(String key, Object value);

    /**
     * 向List结构中添加属性
     */
    Long lPush(String key, Object value, long time);

    /**
     * 向List结构中批量添加属性
     */
    Long lPushAll(String key, Object... values);

    /**
     * 向List结构中批量添加属性
     */
    Long lPushAll(String key, Long time, Object... values);

    /**
     * 从List结构中移除属性
     */
    Long lRemove(String key, long count, Object value);
}
复制代码
```

#### RedisServiceImpl

RedisService的实现类，使用RedisTemplate来自由操作Redis中的缓存数据。

```java
/**
 * redis操作实现类
 * Created by macro on 2020/3/3.
 */
@Service
public class RedisServiceImpl implements RedisService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Override
    public void set(String key, Object value, long time) {
        redisTemplate.opsForValue().set(key, value, time, TimeUnit.SECONDS);
    }

    @Override
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    @Override
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    @Override
    public Boolean del(String key) {
        return redisTemplate.delete(key);
    }

    @Override
    public Long del(List<String> keys) {
        return redisTemplate.delete(keys);
    }

    @Override
    public Boolean expire(String key, long time) {
        return redisTemplate.expire(key, time, TimeUnit.SECONDS);
    }

    @Override
    public Long getExpire(String key) {
        return redisTemplate.getExpire(key, TimeUnit.SECONDS);
    }

    @Override
    public Boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    @Override
    public Long incr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }

    @Override
    public Long decr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, -delta);
    }

    @Override
    public Object hGet(String key, String hashKey) {
        return redisTemplate.opsForHash().get(key, hashKey);
    }

    @Override
    public Boolean hSet(String key, String hashKey, Object value, long time) {
        redisTemplate.opsForHash().put(key, hashKey, value);
        return expire(key, time);
    }

    @Override
    public void hSet(String key, String hashKey, Object value) {
        redisTemplate.opsForHash().put(key, hashKey, value);
    }

    @Override
    public Map<Object, Object> hGetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    @Override
    public Boolean hSetAll(String key, Map<String, Object> map, long time) {
        redisTemplate.opsForHash().putAll(key, map);
        return expire(key, time);
    }

    @Override
    public void hSetAll(String key, Map<String, Object> map) {
        redisTemplate.opsForHash().putAll(key, map);
    }

    @Override
    public void hDel(String key, Object... hashKey) {
        redisTemplate.opsForHash().delete(key, hashKey);
    }

    @Override
    public Boolean hHasKey(String key, String hashKey) {
        return redisTemplate.opsForHash().hasKey(key, hashKey);
    }

    @Override
    public Long hIncr(String key, String hashKey, Long delta) {
        return redisTemplate.opsForHash().increment(key, hashKey, delta);
    }

    @Override
    public Long hDecr(String key, String hashKey, Long delta) {
        return redisTemplate.opsForHash().increment(key, hashKey, -delta);
    }

    @Override
    public Set<Object> sMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    @Override
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }

    @Override
    public Long sAdd(String key, long time, Object... values) {
        Long count = redisTemplate.opsForSet().add(key, values);
        expire(key, time);
        return count;
    }

    @Override
    public Boolean sIsMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }

    @Override
    public Long sSize(String key) {
        return redisTemplate.opsForSet().size(key);
    }

    @Override
    public Long sRemove(String key, Object... values) {
        return redisTemplate.opsForSet().remove(key, values);
    }

    @Override
    public List<Object> lRange(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }

    @Override
    public Long lSize(String key) {
        return redisTemplate.opsForList().size(key);
    }

    @Override
    public Object lIndex(String key, long index) {
        return redisTemplate.opsForList().index(key, index);
    }

    @Override
    public Long lPush(String key, Object value) {
        return redisTemplate.opsForList().rightPush(key, value);
    }

    @Override
    public Long lPush(String key, Object value, long time) {
        Long index = redisTemplate.opsForList().rightPush(key, value);
        expire(key, time);
        return index;
    }

    @Override
    public Long lPushAll(String key, Object... values) {
        return redisTemplate.opsForList().rightPushAll(key, values);
    }

    @Override
    public Long lPushAll(String key, Long time, Object... values) {
        Long count = redisTemplate.opsForList().rightPushAll(key, values);
        expire(key, time);
        return count;
    }

    @Override
    public Long lRemove(String key, long count, Object value) {
        return redisTemplate.opsForList().remove(key, count, value);
    }
}

```



## 引用/参考

[redis三个连接客户端框架的选择：Jedis,Redisson,Lettuce - 斗者_2013 - CSDN](https://blog.csdn.net/w1014074794/article/details/88827946)

[Redis Java Client选型-Jedis Lettuce Redisson - HS_Henry -CSDN](https://blog.csdn.net/chl87783255/article/details/98177040?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.edu_weight&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.edu_weight)

[第三章 Redis 客户端的使用 Java版【Redis入门教程】 - 薛8 -掘金](https://juejin.im/post/5c664b1e51882562914ec10c)

[Spring Data Redis 最佳实践！ - MacroZheng - 掘金](https://juejin.im/post/5e6f703fe51d45270531a214)