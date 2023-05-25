# kafka-consumer KafkaConsumer 源码分析



## KafkaConsumer 属性

org.apache.kafka.clients.consumer.KafkaConsumer

```java
public class KafkaConsumer<K, V> implements Consumer<K, V> {

    private static final Logger log = LoggerFactory.getLogger(KafkaConsumer.class);
    private static final long NO_CURRENT_THREAD = -1L;
    private static final AtomicInteger CONSUMER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);
    private static final String JMX_PREFIX = "kafka.consumer";

    /**
     * Consumer的唯一标识
     */
    private final String clientId;
    /**
     * 控制 consumer 与服务端 groupCoordinator 之间的通信，可以理解为 consumer 与服务端 groupCoordinator 通信的门面
     */
    private final ConsumerCoordinator coordinator;
    /**
     * key 反序列化器
     */
    private final Deserializer<K> keyDeserializer;
    /**
     * value 反序列化器
     */
    private final Deserializer<V> valueDeserializer;
    /**
     * 负责从服务端获取信息
     */
    private final Fetcher<K, V> fetcher;
    /**
     * consumerInterceptor 集合, ConsumerInterceptor#onConsumer 方法可以在消息通过 poll 方法返回给用户之前对其进行拦截或修改
     */
    private final ConsumerInterceptors<K, V> interceptors;

    private final Time time;
    /**
     * 负责消费者和 kafka 服务端的通信
     */
    private final ConsumerNetworkClient client;
    private final Metrics metrics;
    /**
     * 维护了消费者的消费状态
     */
    private final SubscriptionState subscriptions;
    /**
     * 记录了整个 kafka 集群的元信息
     */
    private final Metadata metadata;
    private final long retryBackoffMs;
    private final long requestTimeoutMs;
    private boolean closed = false;

    // currentThread holds the threadId of the current thread accessing KafkaConsumer
    // and is used to prevent multi-threaded access
    /**
     * 记录了当前使用 kafkaConsumer 的 threadId
     * KafkaConsumer#acquire 方法和 release 方法实现了一个"轻量级锁", 并不是真正意义上的锁, 仅仅是为了检测是否有多线程并发操作 KafkaConsumer
     */
    private final AtomicLong currentThread = new AtomicLong(NO_CURRENT_THREAD);
    // refcount is used to allow reentrant access by the thread who has acquired currentThread
    /**
     * 记录了当前使用 kafkaConsumer 的 thread 重入次数
     */
    private final AtomicInteger refcount = new AtomicInteger(0);
    
    // ...
}
```



## KafkaConsumer#poll -- 获取 record

org.apache.kafka.clients.consumer.KafkaConsumer#poll

1. 加锁，出现并发抢锁不会阻塞，而是抛异常。
2. 持续(循环)尝试获取 records，直到获取到了或超时为止。
   1. 尝试获取一次 records。
   2. 获取到 records 则返回，返回前, 如果有 interceptors 则切入。
   3. 未获取到 records，维护剩余超时时间，继续下次循环。
3. finally 中方法返回前解锁。

```java
    @Override
    public ConsumerRecords<K, V> poll(long timeout) {
        // 加锁，出现并发抢锁不会阻塞，而是抛异常
        acquire();
        try {
            if (timeout < 0)
                throw new IllegalArgumentException("Timeout must not be negative");

            // poll for new data until the timeout expires
            // 持续尝试获取 records，直到获取到了或超时为止
            long start = time.milliseconds();
            long remaining = timeout;
            do {
                // 尝试获取一次 records
                Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollOnce(remaining);
                // 获取到 records 则返回，返回前, 如果有 interceptors 则切入
                if (!records.isEmpty()) {
                    // before returning the fetched records, we can send off the next round of fetches
                    // and avoid block waiting for their responses to enable pipelining while the user
                    // is handling the fetched records.
                    //
                    // NOTE: since the consumed position has already been updated, we must not allow
                    // wakeups or any other errors to be triggered prior to returning the fetched records.
                    // Additionally, pollNoWakeup does not allow automatic commits to get triggered.
                    fetcher.sendFetches();
                    client.pollNoWakeup();

                    if (this.interceptors == null)
                        return new ConsumerRecords<>(records);
                    else
                        // 获取到 records 返回前, interceptors 切入点
                        return this.interceptors.onConsume(new ConsumerRecords<>(records));
                }

                long elapsed = time.milliseconds() - start;
                // 未获取到 records，维护剩余超时时间，继续下次循环。
                remaining = timeout - elapsed;
            } while (remaining > 0);

            return ConsumerRecords.empty();
        } finally {
            // 解锁
            release();
        }
    }
```

