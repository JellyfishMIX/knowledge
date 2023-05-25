# kafka-clients Metadata 源码分析



## 说明

1. 本文基于 kafka 2.7 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 概述

元数据对生产者的作用举例: 生产者需要 metadata 感知 broker node 的变化。topic 所在 broker 发生变化时，生产者需要及时知道 topic 分布在哪些 broker 上，否则向错误的 broker 发送消息会失败。

metadata 的获取大致分为: KafkaProducer 线程负责的加载元数据, Sender 线程负责拉取 metadata。



KafkaProducer 的构造方法中初始化了元数据类 MetaData, 然后启动 metadata#bootstrap 引导程序, 此时 metaData 对象里没有具体的元数据信息, 因为客户端还没发送元数据更新的请求。



## Metadata 层次关系

1. Metadata 中封装了元数据的具体信息, 版本控制, 更新等方法。

2. 元数据的信息保存在 MetadataCache 里，MetadataCache 最核心信息是 Cluster，保存了元数据的基础信息。

3. ProducerMetadata 和 ConsumerMetadata 是 Metadata 类的两个子类，分别供给 Producer 和 Consumer 使用。



## Metadata 的属性

1. Metadata 封装了元数据具体的数据和相关操作，会被 KafkaProducer 线程和 Sender 线程使用，是个线程安全的类。
2. Metadata 仅维护着在使用的 topic 元数据，并不是全部的 topic 元数据，能够减少网络 IO 的数据量。如果维护全量 topic, 当集群中有几千个 topic 时, 网络 IO 消耗会很大, 因此生产者只维护涉及自己的 topic 集合的元数据就可以了。

```java
public class Metadata implements Closeable {
    /**
     * 请求元数据失败重试间隔时间，默认 100ms
     */
    private final long refreshBackoffMs;
    /**
     * 元数据过期时间，默认 5 分钟，过期时间一到就发送更新元数据的请求
     */
    private final long metadataExpireMs;
    /**
     * 生产者本地内存元数据版本，每次从服务端获取到元数据就加 1
     */
    private int updateVersion;  // bumped on every metadata response
    /**
     * 每次元数据要加入新的主题都会加一
     */
    private int requestVersion; // bumped on every new topic addition
    /**
     * 最后一次更新元数据的时间
     */
    private long lastRefreshMs;
    /**
     * 最后一次成功更新全部主题元数据的时间
     */
    private long lastSuccessfulRefreshMs;
    private KafkaException fatalException;
    /**
     * 无效的主题
     */
    private Set<String> invalidTopics;
    /**
     * 没有权限的主题
     */
    private Set<String> unauthorizedTopics;
    /**
     * 元数据缓存，客户端真正存储元数据的位置
     */
    private MetadataCache cache = MetadataCache.empty();
    /**
     * 是否需要更新全部 topic
     */
    private boolean needFullUpdate;
    /**
     * 是否需要更新部分 topic
     */
    private boolean needPartialUpdate;
    
    // ...
}
```



## Metadata#bootstrap 方法

org.apache.kafka.clients.Metadata#bootstrap

1. 生产者刚启动，本地缓存中的元数据是空的，所以需要更新全部 topic。updateVersion 为 1。
2. 触发初始化 MetadataCache。

```java
    public synchronized void bootstrap(List<InetSocketAddress> addresses) {
        this.needFullUpdate = true;
        this.updateVersion += 1;
        this.cache = MetadataCache.bootstrap(addresses);
    }
```



因为还没获取到元数据，这时的元数据缓存都是空的数据和集合组成。

向服务器请求元数据在元数据更新器中，元数据类只有解析响应的方法，我们就先看看元数据类是如何解析的，这里又涉及到两个方法——update()和handleMetadataResponse()



## Metadata#update 方法 -- 更新 Metadata

org.apache.kafka.clients.Metadata#update

1. 根据 requestVersion 参数和元数据里的 requestVersion 比较判断是否是更新部分主题的响应。如果是更新全部主题的响应，说明更新全部主题的响应已经收到了, 首先把 needFullUpdate 标记为零, 目的是不要再更新全部主题了, 然后把 lastSuccessfulRefreshMs 更新为当前时间。
2. 处理响应并缓存。



## Metadata#handleMetadataResponse 方法 -- 处理响应并缓存

org.apache.kafka.clients.Metadata#handleMetadataResponse

1. 初始化相关集合。
2. 按主题维度轮询响应中的 TopicMetadata。
3. 判断是否保留当前 topic，有可能 topic 过期了等原因没有必要保留当前 topic。
4. 如果 TopicMetadata 有 error，分别处理不同的 error。
   1. error 如果是无效 metadata，标记需要更新元数据，表示提醒 Sender 线程更新 Metadata。
   2. error 如果是无效 topic，把 topic 放入无效 topic 集合里。
   3. error 如果是 topic 无权限，把 topic 放入无权限 topic 集合里。
5. 如果是部分 topic 的响应, 就和现在的 MetadataCache 整合, 如果不是就重建 MetadataCache。

```java
    private MetadataCache handleMetadataResponse(MetadataResponse metadataResponse, boolean isPartialUpdate, long nowMs) {
        // All encountered topics.
        Set<String> topics = new HashSet<>();

        // Retained topics to be passed to the metadata cache.
        // 初始化相关集合
        Set<String> internalTopics = new HashSet<>();
        Set<String> unauthorizedTopics = new HashSet<>();
        Set<String> invalidTopics = new HashSet<>();

        // 轮询响应中的主题元数据
        List<MetadataResponse.PartitionMetadata> partitions = new ArrayList<>();
        for (MetadataResponse.TopicMetadata metadata : metadataResponse.topicMetadata()) {
            topics.add(metadata.topic());

            // 判断是否保留主题元数据
            if (!retainTopic(metadata.topic(), metadata.isInternal(), nowMs))
                continue;

            // 判断是否是内部主题
            if (metadata.isInternal())
                internalTopics.add(metadata.topic());

            // 元数据响应没有错误就更新本地元数据缓存
            if (metadata.error() == Errors.NONE) {
                // 遍历分区信息
                for (MetadataResponse.PartitionMetadata partitionMetadata : metadata.partitionMetadata()) {
                    // Even if the partition's metadata includes an error, we need to handle
                    // the update to catch new epochs
                    updateLatestMetadata(partitionMetadata, metadataResponse.hasReliableLeaderEpochs())
                        .ifPresent(partitions::add);

                    // 分区数据有问题
                    if (partitionMetadata.error.exception() instanceof InvalidMetadataException) {
                        log.debug("Requesting metadata update for partition {} due to error {}",
                                partitionMetadata.topicPartition, partitionMetadata.error);
                        // 标记需要更新元数据
                        requestUpdate();
                    }
                }
                // 元数据响应有错误
            } else {
                // 无效元数据异常
                if (metadata.error().exception() instanceof InvalidMetadataException) {
                    log.debug("Requesting metadata update for topic {} due to error {}", metadata.topic(), metadata.error());
                    // 标记需要更新元数据
                    requestUpdate();
                }

                // 无效主题的错误
                if (metadata.error() == Errors.INVALID_TOPIC_EXCEPTION)
                    invalidTopics.add(metadata.topic());
                else if (metadata.error() == Errors.TOPIC_AUTHORIZATION_FAILED)
                    unauthorizedTopics.add(metadata.topic());
            }
        }

        Map<Integer, Node> nodes = metadataResponse.brokersById();
        // 如果是部分 topic 的响应, 就和现在的 MetadataCache 整合，如果不是就重建 MetadataCache
        if (isPartialUpdate)
            return this.cache.mergeWith(metadataResponse.clusterId(), nodes, partitions,
                unauthorizedTopics, invalidTopics, internalTopics, metadataResponse.controller(),
                (topic, isInternal) -> !topics.contains(topic) && retainTopic(topic, isInternal, nowMs));
        else
            return new MetadataCache(metadataResponse.clusterId(), nodes, partitions,
                unauthorizedTopics, invalidTopics, internalTopics, metadataResponse.controller());
    }
```

