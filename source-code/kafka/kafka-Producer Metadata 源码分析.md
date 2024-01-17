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

1. Metadata 封装了元数据具体的数据和相关操作。对于生产者，会被 KafkaProducer 线程和 Sender 线程使用，是个线程安全的类。
2. Metadata 仅维护着在使用的 topic 元数据，并不是全部的 topic 元数据，能够减少网络 IO 的数据量。如果维护全量 topic, 当集群中有几千个 topic 时, 网络 IO 消耗会很大, 因此对于生产者和消费者, 只维护涉及自己的 topic 集合的元数据就可以了。

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

### MetadataCache#bootstrap

org.apache.kafka.clients.MetadataCache#bootstrap

此时还没获取到 metadata 信息，缓存信息是空的。

```java
    static MetadataCache bootstrap(List<InetSocketAddress> addresses) {
        Map<Integer, Node> nodes = new HashMap<>();
        int nodeId = -1;
        for (InetSocketAddress address : addresses) {
            nodes.put(nodeId, new Node(nodeId, address.getHostString(), address.getPort()));
            nodeId--;
        }
        return new MetadataCache(null, nodes, Collections.emptyList(),
                Collections.emptySet(), Collections.emptySet(), Collections.emptySet(),
                null, Cluster.bootstrap(addresses));
    }
```



## Metadata#update 方法 -- 更新 Metadata

org.apache.kafka.clients.Metadata#update

1. 根据 requestVersion 参数和元数据里的 requestVersion 比较判断是否是更新部分主题的响应。如果是更新全部主题的响应，说明更新全部主题的响应已经收到了, 首先把 needFullUpdate 标记为零, 目的是不要再更新全部主题了, 然后把 lastSuccessfulRefreshMs 更新为当前时间。
2. 处理响应并缓存。

```java
    public synchronized void update(int requestVersion, MetadataResponse response, boolean isPartialUpdate, long nowMs) {
        Objects.requireNonNull(response, "Metadata response cannot be null");
        if (isClosed())
            throw new IllegalStateException("Update requested after metadata close");
        // 判断部分还是全部 topic 更新，以及更新几个字段
        this.needPartialUpdate = requestVersion < this.requestVersion;
        this.lastRefreshMs = nowMs;
        this.updateVersion += 1;
        if (!isPartialUpdate) {
            this.needFullUpdate = false;
            this.lastSuccessfulRefreshMs = nowMs;
        }

        String previousClusterId = cache.clusterResource().clusterId();

        // 解析元数据响应
        this.cache = handleMetadataResponse(response, isPartialUpdate, nowMs);

        Cluster cluster = cache.cluster();
        maybeSetMetadataError(cluster);

        this.lastSeenLeaderEpochs.keySet().removeIf(tp -> !retainTopic(tp.topic(), false, nowMs));

        String newClusterId = cache.clusterResource().clusterId();
        if (!Objects.equals(previousClusterId, newClusterId)) {
            log.info("Cluster ID: {}", newClusterId);
        }
        clusterResourceListeners.onUpdate(cache.clusterResource());

        log.debug("Updated cluster metadata updateVersion {} to {}", this.updateVersion, this.cache);
    }
```



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



## ProducerMetadata 的属性

```java
public class ProducerMetadata extends Metadata {
    /**
     * 生产者的刷新主题 map，key: topic, value: 此 topic 过期时间
     * 刷新主题 map 的作用是，当 5 分钟 metadata 过期，要向 broker 获取元数据时，仅需要这个 map 里的 topic 对应的 metadata，减少网络 IO 数据量。
     */
    private final Map<String, Long> topics = new HashMap<>();
    /**
     * 第一次发送的 topic 会进入这个集合
     */
    private final Set<String> newTopics = new HashSet<>();
}
```



## ProducerMetadata#add 方法

org.apache.kafka.clients.producer.internals.ProducerMetadata#add

1. 往刷新主题集合里添加这个主题及对应的过期时间(当前时间 + 过期间隔, 默认 5 分钟)。
2. 如果原来不存在这个主题，则把这个主题放到新主题集合中，然后标记需要更新 metadata，等待后续 Sender 线程去请求 broker 获取新主题的 metadata。

```java
    public synchronized void add(String topic, long nowMs) {
        Objects.requireNonNull(topic, "topic cannot be null");
        if (topics.put(topic, nowMs + metadataIdleMs) == null) {
            newTopics.add(topic);
            requestUpdateForNewTopics();
        }
    }
```



## ProducerMetadata#awaitUpdate 方法 -- 阻塞等待 metadata 更新

org.apache.kafka.clients.producer.internals.ProducerMetadata#awaitUpdate

1. 当 KafkaProducer 线程发现没有 topic 对应的 metadata 时，会等待 Sender 线程把 metadata 更新完成。
2. 线程阻塞，底层通过调用 Object.wait() 方法实现了线程的阻塞, 持有 monitor 的对象是当前 ProducerMetadata。

```java
    /**
     * Wait for metadata update until the current version is larger than the last version we know of
     *
     * 当 KafkaProducer 线程发现没有 topic 对应的 metadata 时，会等待 Sender 线程把 metadata 更新完成
     */
    public synchronized void awaitUpdate(final int lastVersion, final long timeoutMs) throws InterruptedException {
        long currentTimeMs = time.milliseconds();
        long deadlineMs = currentTimeMs + timeoutMs < 0 ? Long.MAX_VALUE : currentTimeMs + timeoutMs;
        // 线程阻塞，底层通过调用 Object.wait() 方法实现了线程的阻塞, 持有 monitor 的对象是当前 ProducerMetadata
        time.waitObject(this, () -> {
            // Throw fatal exceptions, if there are any. Recoverable topic errors will be handled by the caller.
            maybeThrowFatalException();
            return updateVersion() > lastVersion || isClosed();
        }, deadlineMs);

        if (isClosed())
            throw new KafkaException("Requested metadata update after close");
    }
```

### SystemTime#waitObject 方法 -- 阻塞等待符合条件

org.apache.kafka.common.utils.SystemTime#waitObject

循环检查是否更新成功, 未成功则阻塞, 更新成功则返回。很经典的阻塞等待符合条件写法。

```java
    @Override
    public void waitObject(Object obj, Supplier<Boolean> condition, long deadlineMs) throws InterruptedException {
        synchronized (obj) {
            // 循环检查是否更新成功，未成功则阻塞
            while (true) {
                // 检查是否更新成功，更新成功则返回
                if (condition.get())
                    return;

                // 超时抛异常
                long currentTimeMs = milliseconds();
                if (currentTimeMs >= deadlineMs)
                    throw new TimeoutException("Condition not satisfied before deadline");

                // 当前线程阻塞
                obj.wait(deadlineMs - currentTimeMs);
            }
        }
    }
```



## ProducerMetadata#update 方法 -- 更新 metadata，唤醒等待元数据更新而阻塞的线程

org.apache.kafka.clients.producer.internals.ProducerMetadata#update

```java
    @Override
    public synchronized void update(int requestVersion, MetadataResponse response, boolean isPartialUpdate, long nowMs) {
        super.update(requestVersion, response, isPartialUpdate, nowMs);

        // Remove all topics in the response that are in the new topic set. Note that if an error was encountered for a
        // new topic's metadata, then any work to resolve the error will include the topic in a full metadata update.
        // 找出已获得 metadata 的相关主题，从新主题集合中删除
        if (!newTopics.isEmpty()) {
            for (MetadataResponse.TopicMetadata metadata : response.topicMetadata()) {
                newTopics.remove(metadata.topic());
            }
        }

        // 唤醒等待元数据更新而阻塞的线程
        notifyAll();
    }
```



## KafkaProducer#doSend -- 阻塞等待 metadata 更新的调用处

org.apache.kafka.clients.producer.KafkaProducer#doSend

KafkaProducer 在发送消息前尝试阻塞等待 metadata 更新(如无需更新 metadata 则不阻塞)。

```java
    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;
        try {
            throwIfProducerClosed();
            // first make sure the metadata for the topic is available
            long nowMs = time.milliseconds();
            ClusterAndWaitTime clusterAndWaitTime;
            try {
                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
            } catch (KafkaException e) {
                if (metadata.isClosed())
                    throw new KafkaException("Producer closed while send in progress", e);
                throw e;
            }
            nowMs += clusterAndWaitTime.waitedOnMetadataMs;
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
        // ...
    }
```

### KafkaProducer#waitOnMetadata -- 尝试阻塞等待 metadata 更新

org.apache.kafka.clients.producer.KafkaProducer#waitOnMetadata

1. 获取生产者缓存的 cluster 信息，判断 topic 是否是有效，如果无效则放入无效主题集合中。
2. 唤醒 Sender 线程。Sender 线程一直在运行，唤醒的目的是让 Sender 线程从 select() 阻塞中立即返回，然后把获取元数据 Channel 的事件注册在 Selector 上，这样就可以及时监听获取元数据的事件了。
3. 如果客户端缓存中的元数据能找到消息发送对应分区，就不用去服务端请求更新元数据了，直接返回从生产者缓存中的元数据。大部分消息在这里会命中缓存。
4. 循环持续等待 Sender 获取更新 metadata。
   1. 标记元数据需要更新，并获得版本。
   2. 唤醒 Sender 线程。当前线程阻塞等待 metadata 更新。
   3. 维护分区信息。
5. 返回获取的元数据和获取元数据消耗的时间。

```java
    private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long nowMs, long maxWaitMs) throws InterruptedException {
        // add topic to metadata topic list if it is not there already and reset expiry
        // 获取 cluster 信息
        Cluster cluster = metadata.fetch();

        // 判断 topic 是否无效
        if (cluster.invalidTopics().contains(topic))
            throw new InvalidTopicException(topic);

        // 把主题加入 metadata
        metadata.add(topic, nowMs);

        // 从 cluster 中找到 topic 对应分区数
        Integer partitionsCount = cluster.partitionCountForTopic(topic);
        // Return cached metadata if we have it, and if the record's partition is either undefined
        // or within the known partition range
        // 如果客户端缓存中的元数据能找到消息发送对应分区，就不用去服务端请求更新元数据了，直接返回从生产者缓存中的元数据。大部分消息在这里会命中缓存。
        if (partitionsCount != null && (partition == null || partition < partitionsCount))
            return new ClusterAndWaitTime(cluster, 0);

        long remainingWaitMs = maxWaitMs;
        long elapsed = 0;
        // Issue metadata requests until we have metadata for the topic and the requested partition,
        // or until maxWaitTimeMs is exceeded. This is necessary in case the metadata
        // is stale and the number of partitions for this topic has increased in the meantime.
        // 循环持续等待 Sender 获取更新 metadata
        do {
            if (partition != null) {
                log.trace("Requesting metadata update for partition {} of topic {}.", partition, topic);
            } else {
                log.trace("Requesting metadata update for topic {}.", topic);
            }
            // 把主题和过期时间加入元数据主题列表中
            metadata.add(topic, nowMs + elapsed);
            // 标记元数据需要更新，并获得版本
            int version = metadata.requestUpdateForTopic(topic);
            // 唤醒 Sender 线程
            sender.wakeup();
            try {
                // 当前线程阻塞等待 metadata 更新
                metadata.awaitUpdate(version, remainingWaitMs);
            } catch (TimeoutException ex) {
                // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
                throw new TimeoutException(
                        String.format("Topic %s not present in metadata after %d ms.",
                                topic, maxWaitMs));
            }
            // 获取元数据
            cluster = metadata.fetch();
            // 计算等待更新元数据消耗了多少时间
            elapsed = time.milliseconds() - nowMs;
            // 超时抛出异常
            if (elapsed >= maxWaitMs) {
                throw new TimeoutException(partitionsCount == null ?
                        String.format("Topic %s not present in metadata after %d ms.",
                                topic, maxWaitMs) :
                        String.format("Partition %d of topic %s with partition count %d is not present in metadata after %d ms.",
                                partition, topic, partitionsCount, maxWaitMs));
            }
            metadata.maybeThrowExceptionForTopic(topic);
            remainingWaitMs = maxWaitMs - elapsed;
            // 获取元数据分区数
            partitionsCount = cluster.partitionCountForTopic(topic);
        } while (partitionsCount == null || (partition != null && partition >= partitionsCount));

        // 返回获取的元数据和获取元数据消耗的时间
        return new ClusterAndWaitTime(cluster, elapsed);
    }
```



## MetadataUpdater -- 元数据更新接口

MetadataUpdater 接口提供了元数据更新的能力，客户端的实现是 DefaultMetadataUpdater 类。

### DefaultMetadataUpdater#maybeUpdate(long) 方法 -- 尝试更新 metadata

org.apache.kafka.clients.NetworkClient.DefaultMetadataUpdater#maybeUpdate(long)

1. 根据当前时间和退避时间(防止请求过于频繁而设置的间隔时间)，计算下一次 Metadata 更新的时间戳。

```java
        /**
         * 尝试更新 metadata, 用于正式发送获取元数据请求前的判断, 主要是判断发送元数据请求的时机
         */
        @Override
        public long maybeUpdate(long now) {
            // should we update our metadata?
            // 下次更新的时间
            long timeToNextMetadataUpdate = metadata.timeToNextUpdate(now);
            // 检测是否已经发送了 MetadataRequest 请求
            long waitForMetadataFetch = hasFetchInProgress() ? defaultRequestTimeoutMs : 0;

            long metadataTimeout = Math.max(timeToNextMetadataUpdate, waitForMetadataFetch);
            if (metadataTimeout > 0) {
                return metadataTimeout;
            }

            // Beware that the behavior of this method and the computation of timeouts for poll() are
            // highly dependent on the behavior of leastLoadedNode.
            // 找到最小负载的 node
            Node node = leastLoadedNode(now);
            if (node == null) {
                log.debug("Give up sending metadata request since no node is available");
                return reconnectBackoffMs;
            }

            return maybeUpdate(now, node);
        }
```

