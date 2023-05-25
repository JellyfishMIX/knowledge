# kafka-consumer Fetcher 源码分析



## 简介

1. Fetcher 用于从 broker 获取消息。
2. Fetcher 类的主要功能是发送 FetchRequest 请求，获取指定的消息集合，处理 FetchResponse，更新 offset。

```java
public class Fetcher<K, V> {	
	/**
     * consumer 网络通信组件
     */
    private final ConsumerNetworkClient client;
    private final Time time;
    /**
     * broker 收到 FetchRequest 后不是立即响应，而是当可返回的消息数据积累到至少 minBytes 个字节时才响应。这样可以批量返回更多的消息，提高网络 IO 的效率。
     */
    private final int minBytes;
    /**
     * broker 响应 FetchResponse 的最长等待时间
     */
    private final int maxWaitMs;
    /**
     * 每次 fetch 操作的最大字节数
     */
    private final int fetchSize;
    private final long retryBackoffMs;
    /**
     * 每次获取 record 的最大数量
     */
    private final int maxPollRecords;
    private final boolean checkCrcs;
    /**
     * kafka 集群的元数据
     */
    private final Metadata metadata;
    private final FetchManagerMetrics sensors;
    /**
     * 记录每个 TopicPartition 的消费情况
     */
    private final SubscriptionState subscriptions;
    /**
     * 暂未解析的消息。每个 FetchResponse 首先会转换成 CompletedFetch 对象进入此队列缓存
     */
    private final List<CompletedFetch> completedFetches;
    /**
     * key 的反序列化器
     */
    private final Deserializer<K> keyDeserializer;
    /**
     * value 的反序列化器
     */
    private final Deserializer<V> valueDeserializer;

    /**
     * PartitionRecords 保存了 CompletedFetch 解析后的结果集合，其中有三个字段: records 是消息集合，fetchOffset 记录了 records 中第一个消息的 offset, partition 记录了消息对应的 TopicPartition
     */
    private PartitionRecords<K, V> nextInLineRecords = null;
    
    // ...
}
```



## Fetcher#createFetchRequests 方法 -- 创建 FetchRequest 请求

创建 FetchRequest 请求, 返回值是 key 是 Node, value 是发往对应 Node 的 FetchRequest 集合

org.apache.kafka.clients.consumer.internals.Fetcher#createFetchRequests

1. 获取kafka集群数据。
2. fetchablePartitions 方法，按照一定的条件过滤后得到可以发送 FetchRequest 的分区。
3. 查找分区的 leader 副本所在的 node。如果找不到 leader 副本则更新 metadata。
4. 如果发送给 leader node 还有 pending 请求，则记录分区的对应的 position, 即要 fetch 消息的 offset
5. 对上面的 fetchable 集合进行转换，将发往同一个 node 节点的所有 TopicPartition 的 position 信息封装成一个 FetchRequest 对象。

```java
	private Map<Node, FetchRequest> createFetchRequests() {
        // create the fetch info
        // 获取 kafka 集群数据
        Cluster cluster = metadata.fetch();
        Map<Node, Map<TopicPartition, FetchRequest.PartitionData>> fetchable = new HashMap<>();
        // fetchablePartitions 方法，按照一定的条件过滤后得到可以发送 FetchRequest 的分区
        for (TopicPartition partition : fetchablePartitions()) {
            // 查找分区的 leader 副本所在的 node
            Node node = cluster.leaderFor(partition);
            // 如果找不到 leader 副本则更新 metadata
            if (node == null) {
                metadata.requestUpdate();
                // 是否还有 pending 请求
            } else if (this.client.pendingRequestCount(node) == 0) {
                // if there is a leader and no in-flight requests, issue a new fetch
                Map<TopicPartition, FetchRequest.PartitionData> fetch = fetchable.get(node);
                if (fetch == null) {
                    fetch = new HashMap<>();
                    fetchable.put(node, fetch);
                }

                long position = this.subscriptions.position(partition);
                // 记录当前分区对应的 position, 即要 fetch 消息的 offset
                fetch.put(partition, new FetchRequest.PartitionData(position, this.fetchSize));
                log.trace("Added fetch request for partition {} at offset {}", partition, position);
            }
        }

        // create the fetches
        // 对上面的 fetchable 集合进行转换，将发往同一个 node 节点的所有 TopicPartition 的 position 信息封装成一个 FetchRequest 对象
        Map<Node, FetchRequest> requests = new HashMap<>();
        for (Map.Entry<Node, Map<TopicPartition, FetchRequest.PartitionData>> entry : fetchable.entrySet()) {
            Node node = entry.getKey();
            FetchRequest fetch = new FetchRequest(this.maxWaitMs, this.minBytes, entry.getValue());
            requests.put(node, fetch);
        }
        return requests;
    }
```



## Fetcher#sendFetches 方法 -- 缓存 FetchRequest 等待发送

org.apache.kafka.clients.consumer.internals.Fetcher#sendFetches

1. 将待发送的请求封装成 ClientRequest，然后保存在 unsent 集合中等待发送。
2. 给请求结果 future 添加 RequestFutureListener，用于处理 FetchResponse。遍历每个分区的响应，创建 CompletedFetch，将获得到的消息数据(未解析的byte数组)和 offset 组装成 CompletedFetch 对象，加入 CompletedFetches 队列中。

```java
	public void sendFetches() {
        for (Map.Entry<Node, FetchRequest> fetchEntry: createFetchRequests().entrySet()) {
            final FetchRequest request = fetchEntry.getValue();
            // 将待发送的请求封装成 ClientRequest，然后保存在 unsent 集合中等待发送
            client.send(fetchEntry.getKey(), ApiKeys.FETCH, request)
                    // 给请求结果 future 添加 RequestFutureListener，这是处理 FetchResponse 的入口
                    .addListener(new RequestFutureListener<ClientResponse>() {
                        @Override
                        public void onSuccess(ClientResponse resp) {
                            FetchResponse response = new FetchResponse(resp.responseBody());
                            Set<TopicPartition> partitions = new HashSet<>(response.responseData().keySet());
                            FetchResponseMetricAggregator metricAggregator = new FetchResponseMetricAggregator(sensors, partitions);
                            // 遍历每个分区的响应
                            for (Map.Entry<TopicPartition, FetchResponse.PartitionData> entry : response.responseData().entrySet()) {
                                TopicPartition partition = entry.getKey();
                                long fetchOffset = request.fetchData().get(partition).offset;
                                FetchResponse.PartitionData fetchData = entry.getValue();
                                // 创建 CompletedFetch，并加入 CompletedFetches 队列中
                                completedFetches.add(new CompletedFetch(partition, fetchOffset, fetchData, metricAggregator));
                            }

                            sensors.fetchLatency.record(resp.requestLatencyMs());
                            sensors.fetchThrottleTimeSensor.record(response.getThrottleTime());
                        }

                        @Override
                        public void onFailure(RuntimeException e) {
                            log.debug("Fetch failed", e);
                        }
                    });
        }
    }
```



## Fetcher#fetchedRecords 方法 -- 获取并解析 records

org.apache.kafka.clients.consumer.internals.Fetcher#fetchedRecords

存储在 CompletedFetches 队列中的数据是未解析的 FetchResponse, 此方法解析得到 records 集合并返回。同时修改对应的分区的 position, 为下次操作做准备。

具体逻辑:

1. 需要进行 Rebalance 操作则返回空集合。
2. 声明返回结果，按照 TopicPartition 分类 records。
3. 维护剩余待获取的 record 条数，一次最多取出 maxPollRecords 条消息。
   1. 遍历 completedFetches 集合并解析获得 records。
   2. 遍历过程中维护 nextInLineRecords, 是解析好的 records，如果有已解析好的 records 则添加至返回结果中，没有已解析好的则现解析。

```java
	/**
     * 存储在 CompletedFetches 队列中的数据是未解析的 FetchResponse, 此方法解析得到 records 集合并返回。
     * 同时修改对应的分区的 position, 为下次操作做准备。
     */
    public Map<TopicPartition, List<ConsumerRecord<K, V>>> fetchedRecords() {
        if (this.subscriptions.partitionAssignmentNeeded()) {
            // 需要进行 Rebalance 操作则返回空集合
            return Collections.emptyMap();
        } else {
            // 声明返回结果，按照 TopicPartition 分类 records
            Map<TopicPartition, List<ConsumerRecord<K, V>>> drained = new HashMap<>();
            // 剩余待获取的 record 条数，一次最多取出 maxPollRecords 条消息
            int recordsRemaining = maxPollRecords;
            // completedFetches 集合的迭代器
            Iterator<CompletedFetch> completedFetchesIterator = completedFetches.iterator();

            // 遍历 completedFetches 集合并解析
            while (recordsRemaining > 0) {
                // nextInLineRecords 是解析好的 records，如果有已解析好的 records 则添加至返回结果中，没有已解析好的则现解析
                if (nextInLineRecords == null || nextInLineRecords.isEmpty()) {
                    if (!completedFetchesIterator.hasNext())
                        break;

                    CompletedFetch completion = completedFetchesIterator.next();
                    completedFetchesIterator.remove();
                    nextInLineRecords = parseFetchedData(completion);
                } else {
                    recordsRemaining -= append(drained, nextInLineRecords, recordsRemaining);
                }
            }

            return drained;
        }
    }
```

