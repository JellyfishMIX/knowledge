## kafka-consumer SubscriptionState 源码分析



## 说明

1. 本文基于 kafka 2.7 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## SubscriptionState 的属性

consumer 使用 SubscriptionState 保存消费者订阅的主题，关联 TopicPartition 与 offset。

1. groupSubscription
   1. 当前消费者如果是 ConsumerCoordinator leader，这个字段存储消费组所有消费者订阅的主题，用于监控消费组主题相关的元数据的变化，以便 rebalance 时为消费组制定分区方案。
   2. 当前消费者如果是 ConsumerCoordinator follower，由于不涉及为消费组全体消费者指定分区方案，这个字段只会保存本消费者订阅的主题。

```java
    /**
     * 表示订阅 topic 的模式
     */
    private enum SubscriptionType {
        NONE,
        // 按指定的 topic 进行订阅，自动分配分区
        AUTO_TOPICS,
        // 按正则表达式匹配的 topic 进行订阅，自动分配分区
        AUTO_PATTERN,
        // 用户自己定制消费者要消费的 topic 和分区
        USER_ASSIGNED
    }

    /* the type of subscription */
    private SubscriptionType subscriptionType;

    /**
     * 用来过滤 topic 的正则表达式
     */
    /* the pattern user has requested */
    private Pattern subscribedPattern;

    /**
     * 用户手动写的要订阅的 topic
     */
    /* the list of topics the user has requested */
    private Set<String> subscription;

    /**
     * 消费组订阅的所有 topic
     */
    /* The list of topics the group has subscribed to. This may include some topics which are not part
     * of `subscription` for the leader of a group since it is responsible for detecting metadata changes
     * which require a group rebalance. */
    private Set<String> groupSubscription;

    /**
     * 记录消费者里主题分区的状态集合
     */
    /* the partitions that are currently assigned, note that the order of partition matters (see FetchBuilder for more details) */
    private final PartitionStates<TopicPartitionState> assignment;

    /**
     * 默认重置策略
     * 当消费者重启时使用的策略，消费策略有两种: LATEST, 从分区最后位置消费。EARLIEST, 从分区最一开始的位置消费。
     */
    /* Default offset reset strategy */
    private final OffsetResetStrategy defaultResetStrategy;

    /**
     * 用于监听 rebalance 后消费者要消费的分区变化
     */
    /* User-provided listener to be invoked when assignment changes */
    private ConsumerRebalanceListener rebalanceListener;
```



## SubscriptionState#subscribe 方法 -- 用户指定订阅的主题

org.apache.kafka.clients.consumer.internals.SubscriptionState#subscribe

1. 注册 ConsumerRebalanceListener。
2. 设置订阅模式 AUTO_TOPICS, 按指定的 topic 进行订阅，自动分配分区。
3. 配置订阅的主题。

```java
    /**
     * 用户指定订阅的主题
     */
    public synchronized boolean subscribe(Set<String> topics, ConsumerRebalanceListener listener) {
        // 注册 ConsumerRebalanceListener
        registerRebalanceListener(listener);
        // 设置订阅模式 AUTO_TOPICS, 按指定的 topic 进行订阅，自动分配分区
        setSubscriptionType(SubscriptionType.AUTO_TOPICS);
        // 配置订阅的主题
        return changeSubscription(topics);
    }
```



## OffsetCommitRequest 和 OffsetCommitResponse 的消息体格式

### OffsetCommitRequest

字段含义:

| 名称                | 类型   | 说明                                   |
| ------------------- | ------ | -------------------------------------- |
| group_id            | String | ConsumerGroup 的 id                    |
| group_generation_id | int    | 消费者保存的年代信息                   |
| member_id           | String | GroupCoordinator 分配给 consumer 的 id |
| retention_time      | long   | 此 offset 的最长保存时间               |
| topic               | String | topic 名称                             |
| partition           | int    | 分区号                                 |
| offset              | long   | 提交的 offset                          |
| metadata            | String | 任何希望与 offset 一起保存的自定义数据 |



![E016F3AC-289E-41BA-BA2F-A35424622D9B.png](https://image-hosting.jellyfishmix.com/20230613210313.png)

### OffsetCommitResponse

字段含义:

| 名称       | 类型   | 说明       |
| ---------- | ------ | ---------- |
| topic      | String | topic 名称 |
| partition  | int    | 分区编号   |
| error_code | short  | 错误码     |

![E4C837C3-749B-4DE6-83F5-896221E8B768.png](https://image-hosting.jellyfishmix.com/20230613211113.png)



## consumer 提交 offset 的时机

1. 发生 rebalance 时，消费者要向 broker GroupCoordinator 提交 offset 记录当前消费的位置。
2. consumer 消费过程中主动向 broker 提交 offset, 提交方式有异步和同步两种。

### 提交方式

1. 异步提交。
   1. consumer 线程提交 offset 时，不用等待 broker 的响应，consumer 线程可以继续执行后续逻辑。响应结果由 callback 处理。
   2. 提高了 consumer 线程的性能，适合对感知是否成功消费不敏感的业务。
2. 同步提交。
   1. consumer 线程提交 offset 时，需等待服务端的响应。这种场景会造成线程的阻塞，影响线程的并发性，但是适用于线程后面的逻辑对消费者是否成功消费敏感。
   2. 存在阻塞，consumer 线程的性能会降低，但可以保证感知向 broker 提交 offset 是否成功后才进行下次消费，适合对感知是否成功消费敏感的业务。

本文只分析异步提交。



## ConsumerCoordinator#maybeAutoCommitOffsetsAsync 方法 -- 尝试自动提交 offset

org.apache.kafka.clients.consumer.internals.ConsumerCoordinator#maybeAutoCommitOffsetsAsync

1. 检查是否开启了 offset 自动提交。
2. 记录当前 offset 提交时间，目的是为了计时。
3. 到达下次自动提交的时间，则进行 offset 提交。自动提交时间配置是 auto.commit.interval.ms，默认值 5000ms。
4. 重置下次 offset 自动提交时间。

```java
    public void maybeAutoCommitOffsetsAsync(long now) {
        // 是否开启了 offset 自动提交
        if (autoCommitEnabled) {
            // 记录当前 offset 提交时间，目的是为了计时
            nextAutoCommitTimer.update(now);
            // 到达下次自动提交的时间，进行 offset 提交
            if (nextAutoCommitTimer.isExpired()) {
                // 重置下次 offset 自动提交时间
                nextAutoCommitTimer.reset(autoCommitIntervalMs);
                doAutoCommitOffsetsAsync();
            }
        }
    }
```

