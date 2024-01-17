# kafka-Producer 核心流程分析



## 说明

1. 本文基于 kafka 0.10.0 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## KafkaProducer

org.apache.kafka.clients.producer.KafkaProducer

```java
public class KafkaProducer<K, V> implements Producer<K, V> {
    private static final Logger log = LoggerFactory.getLogger(KafkaProducer.class);
    // 用于生产者客户端名称的生成，自增序列器
    private static final AtomicInteger PRODUCER_CLIENT_ID_SEQUENCE = new AtomicInteger(1);
    // JMX 中显示的前缀
    private static final String JMX_PREFIX = "kafka.producer";
    // 生产者客户端的名称
    private String clientId;
    // 分区器
    private final Partitioner partitioner;
    // 消息的最大长度，包含了消息头、序列化后的 key 和序列化后的 value 的长度
    private final int maxRequestSize;
    // 发送单个消息的缓冲区大小
    private final long totalMemorySize;
    // 集群元数据信息
    private final Metadata metadata;
    // 用于存放消息的缓冲
    private final RecordAccumulator accumulator;
    // 发送消息的 sender 任务
    private final Sender sender;
    // 性能监控相关
    private final Metrics metrics;
    // 发送消息的线程，sender 对象会在该线程中运行
    private final Thread ioThread;
    // 压缩算法
    private final CompressionType compressionType;
    // 错误记录器
    private final Sensor errors;
    // 用于时间相关操作
    private final Time time;
    // 键和值序列化器
    private final Serializer<K> keySerializer;
    private final Serializer<V> valueSerializer;
    // 生产者配置集
    private final ProducerConfig producerConfig;
    // 等待更新 kafka 集群元数据的最大时长
    private final long maxBlockTimeMs;
    // 消息的超时时间，也就是从消息发送到收到 ACK 响应的最长时长
    private final int requestTimeoutMs;
    // 拦截器集合
    private final ProducerInterceptors<K, V> interceptors;
}
```



## KafkaProducer#send 方法 -- 发送消息

org.apache.kafka.clients.producer.KafkaProducer#send

producer发送数据都是调用 send 方法。

```java
    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
        return send(record, null);
    }
    
    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        // 使用 ProducerInterceptor 可对消息进行拦截处理
        ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
        // 发送消息核心方法
        return doSend(interceptedRecord, callback);
    }
```



## KafkaProducer#doSend 方法 -- 发送消息核心方法

org.apache.kafka.clients.producer.KafkaProducer#doSend

发送消息核心方法。

1. 确保目标 topic 的 metadata 处于可用状态，maxBlockTimeMs 最多能等待多久。
2. 对消息的 key 和 value 进行序列化。
3. 根据分区器选择消息应该发送的分区。前面已经获取到了元数据，这里根据元数据的信息计算应该把当前数据发送到哪个分区。
4. 计算消息记录的总大小，确认消息的大小是否超过了最大值，最大值默认 1M，可配置。
5. 根据元数据信息，封装分区对象。
6. 给消息绑定回调函数，用来支持异步的方式发送的消息。
7. 向分区 RecordBatch 队列中追加消息。
8. 如果达到批次要求，唤醒 sender 线程，这才是真正发送数据的线程。

```java
    private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;
        try {
            // first make sure the metadata for the topic is available
            // 确保目标 topic 的 metadata 处于可用状态，maxBlockTimeMs 最多能等待多久。
            ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
            // 对消息的 key 和 value 进行序列化
            byte[] serializedKey;
            try {
                serializedKey = keySerializer.serialize(record.topic(), record.key());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in key.serializer");
            }
            byte[] serializedValue;
            try {
                serializedValue = valueSerializer.serialize(record.topic(), record.value());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in value.serializer");
            }

            // 根据分区器选择消息应该发送的分区。前面已经获取到了元数据，这里根据元数据的信息计算应该把当前数据发送到哪个分区。
            int partition = partition(record, serializedKey, serializedValue, cluster);
            // 计算消息记录的总大小
            int serializedSize = Records.LOG_OVERHEAD + Record.recordSize(serializedKey, serializedValue);
            // 确认消息的大小是否超过了最大值，最大值默认 1M，可配置
            ensureValidRecordSize(serializedSize);
            // 根据元数据信息，封装分区对象
            tp = new TopicPartition(record.topic(), partition);
            long timestamp = record.timestamp() == null ? time.milliseconds() : record.timestamp();
            log.trace("Sending record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
            // producer callback will make sure to call both 'callback' and interceptor callback
            // 给消息绑定回调函数，用来支持异步的方式发送的消息
            Callback interceptCallback = this.interceptors == null ? callback : new InterceptorCallback<>(callback, this.interceptors, tp);
            // 向分区 RecordBatch 队列中追加消息
            RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, interceptCallback, remainingWaitMs);
            // 如果达到批次要求，唤醒 sender 线程，这才是真正发送数据的线程
            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
                this.sender.wakeup();
            }
            return result.future;
            // handling exceptions and record the errors;
            // for API exceptions return them in the future,
            // for other exceptions throw directly
        } catch (ApiException e) {
            log.debug("Exception occurred during message send:", e);
            if (callback != null)
                callback.onCompletion(null, e);
            this.errors.record();
            if (this.interceptors != null)
                this.interceptors.onSendError(record, tp, e);
            return new FutureFailure(e);
        } catch (InterruptedException e) {
            this.errors.record();
            if (this.interceptors != null)
                this.interceptors.onSendError(record, tp, e);
            throw new InterruptException(e);
        } catch (BufferExhaustedException e) {
            this.errors.record();
            this.metrics.sensor("buffer-exhausted-records").record();
            if (this.interceptors != null)
                this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (KafkaException e) {
            this.errors.record();
            if (this.interceptors != null)
                this.interceptors.onSendError(record, tp, e);
            throw e;
        } catch (Exception e) {
            // we notify interceptor about all exceptions, since onSend is called before anything else in this method
            if (this.interceptors != null)
                this.interceptors.onSendError(record, tp, e);
            throw e;
        }
    }
```

### 对消息的 key 和 value 序列化

ProducerRecord 消息的 key 和 value 可以是各种各样的类型，比如 String, Double, Boolean，或者是自定义的对象。如果要发送消息到 broker，必须对 key 和 value 进行序列化，把各种类型的数据类型转换为 byte[] 字节数组的形式。kafka 内部提供的序列化和反序列化组件如下。如果这些序列化组件不能满足需求，可以自定义 Serializer 的实现。

org.apache.kafka.common.serialization.Serializer

![image-20230309110144016](https://image-hosting.jellyfishmix.com/20230309110144.png)

### 为消息选择分区 partition

org.apache.kafka.clients.producer.internals.DefaultPartitioner#partition

Producer 默认使用的分区器是 org.apache.kafka.clients.producer.internals.DefaultPartitioner, 用户也可以自定义实现。

1. 获取集群中指定 topic 的分区信息。
2. 策略一: 如果消息中没有指定 key，则轮转分区，获取 counter 并自增，counter 是个原子类。
3. 策略二: 策略二: 如果消息中指定了 key, 直接对 key 取一个 hash 值 % 分区的总数取模。如果是同一个 key，计算出来的分区肯定是同一个分区。murmur2 是一种高效率低碰撞的 hash 算法。

```java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 获取集群中指定 topic 的分区信息
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            // 策略一: 如果消息中没有指定 key, 轮转
            // 获取 counter 并自增，counter 是个原子类
            int nextValue = nextValue(topic);
            // 获取可用分区
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                // 没有可用分区，直接给一个不可用分区
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            /*
             * hash the keyBytes to choose a partition
             * 策略二: 如果消息中指定了 key, 直接对 key 取一个 hash 值 % 分区的总数取模
             * 如果是同一个 key，计算出来的分区肯定是同一个分区
             * murmur2 是一种高效率低碰撞的 hash 算法
             */
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

### 校验消息大小是否超过最大值

1. 如果一条消息的大小超过最大请求大小(默认 1M 可配置)会抛出异常。
2. 如果一条消息的大小超过 producer 可用内存(默认 32M 可配置)会抛出异常。
3. max.request.size 和 buffer.memory 均可配置。

```java
	private void ensureValidRecordSize(int size) {
        // 如果一条消息的大小超过最大请求大小(默认 1M 可配置)会抛出异常
        if (size > this.maxRequestSize)
            throw new RecordTooLargeException("The message is " + size +
                                              " bytes when serialized which is larger than the maximum request size you have configured with the " +
                                              ProducerConfig.MAX_REQUEST_SIZE_CONFIG +
                                              " configuration.");
        // 如果一条消息的大小超过 producer 可用内存(默认 32M 可配置)会抛出异常
        if (size > this.totalMemorySize)
            throw new RecordTooLargeException("The message is " + size +
                                              " bytes when serialized which is larger than the total memory buffer you have configured with the " +
                                              ProducerConfig.BUFFER_MEMORY_CONFIG +
                                              " configuration.");
    }
```

### RecordAccumulator#append 方法 -- 向分区 RecordBatch 队列中追加消息

org.apache.kafka.clients.producer.internals.RecordAccumulator#append

每个分区有一个 RecordBatch 队列，表示要向此分区发送的消息。

1. 统计正在向 RecordAccumulator 中追加数据的线程数。
2. 先根据分区找到应该插入到哪个队列，每个分区一个队列。
3. synchronized 线程安全区，锁 monitor 对象为分区队列。
   1. 检查生产者是否已经关闭了。
   2. 尝试往队列里的批次添加数据，第一次添加数据肯定是失败的。
   3. 数据需要存储在 RecordBatch 中(需要分配内存)，第一次添加数据时还没有分配内存，结果会是 null
4. 在消息的大小和批次的大小之间，取最大值作为当前批次的大小。有可能一个消息的大小比设定好的批次的大小(默认 16K)还要大。
5. 根据 size 分配内存，线程安全地创建一个 RecordBatch。synchronized 线程安全区，锁 monitor 对象为分区队列。
   1. 第一次添加数据依然失败，虽然分配了内存，但还没有创建 RecordBatch，不能向 RecordBatch 中写数据。
   2. 非第一次添加数据的线程 tryAppend 结果不为 null，把数据写到批次了，释放内存(还给内存池)。
   3. 第一次添加数据失败后，根据内存封装 RecordBatch
   4. 向 RecordBatch 中写具体消息数据，把 RecordBatch 追加至当前队列的队尾。
6. try finally 中，将正在追加消息线程数的计数器 -1

```java
    public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        // 统计正在向 RecordAccumulator 中追加数据的线程数
        appendsInProgress.incrementAndGet();
        try {
            // check if we have an in-progress batch
            // 先根据分区找到应该插入到哪个队列，每个分区一个队列
            Deque<RecordBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                // 检查生产者是否已经关闭了
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");
                /*
                 * 尝试往队列里的批次添加数据，第一次添加数据肯定是失败的。
                 * 数据需要存储在 RecordBatch 中(需要分配内存)，第一次添加数据时还没有分配内存，结果会是 null
                 */
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            // we don't have an in-progress record batch try to allocate a new batch
            // 在消息的大小和批次的大小之间，取最大值作为当前批次的大小。有可能一个消息的大小比设定好的批次的大小(默认 16K)还要大
            int size = Math.max(this.batchSize, Records.LOG_OVERHEAD + Record.recordSize(key, value));
            log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
            // 根据 size 分配内存
            ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
            // 线程安全地创建一个 RecordBatch
            synchronized (dq) {
                // Need to check if producer is closed again after grabbing the dequeue lock.
                if (closed)
                    throw new IllegalStateException("Cannot send after the producer is closed.");

                // 第一次添加数据依然失败，虽然分配了内存，但还没有创建 RecordBatch，不能向 RecordBatch 中写数据
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, callback, dq);
                if (appendResult != null) {
                    // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                    // 非第一次添加数据的线程 tryAppend 结果不为 null，把数据写到批次了，释放内存(还给内存池)
                    free.deallocate(buffer);
                    return appendResult;
                }
                // 第一次添加数据失败后，根据内存封装 RecordBatch
                MemoryRecordsBuilder recordsBuilder = MemoryRecords.builder(buffer, compression, TimestampType.CREATE_TIME, this.batchSize);
                RecordBatch batch = new RecordBatch(tp, recordsBuilder, time.milliseconds());
                // 向 RecordBatch 中写具体消息数据
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, callback, time.milliseconds()));

                // 把 RecordBatch 追加至当前队列的队尾
                dq.addLast(batch);
                incomplete.add(batch);
                return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
            }
        } finally {
            // 正在追加消息线程数的计数器 -1
            appendsInProgress.decrementAndGet();
        }
    }
```

### producer 内存池设计

1. batches 里面存储的是 RecordBatch，批次默认的大小是 16K，整个缓存的大小是 32M，生产者每封装一个批次都需要去申请内存。
2. 按照普通设计思路，一个 RecordBatch 发送出去后，这 16K 的内存等待 GC 回收，这样可能会频繁 GC，影响生产者的性能。
3. 所以 kafka 设计了一个内存池(类似于数据库的连接池，池化思想为了复用资源)，一个 16K 的内存用完后，把数据清空，放入到内存池中。下个 RecordBatch 用时直接从内存池中获取就可以。这样减少了 GC 的频率，提升了 producer 的性能。

### BufferPool#allocate -- 在内存池中分配指定 size 的内存

org.apache.kafka.clients.producer.internals.BufferPool#allocate

1. 分配内存前对 bufferPool 加锁。
2. 如果申请的资源正好是 BATCH.SIZE 的大小，并且 bufferPool.free 链表中有可用的 ByteBuffer，直接使用。
3. bufferPool 可用内存大小 = this.nonPooledAvailableMemory + freeListSize，判断是否足够本次需要分配的 size。
4. bufferPool 可用资源够用时，释放掉一些 bufferPool.free 中的 byteBuffer，确保够 size 使用。
   1. 解锁，返回分配的内存。
5. bufferPool 可用资源不够时，等待资源。循环以下逻辑直到内存够用。
   1. 等待有内存资源释放。(如果等待超时则报错)
   2. 上面等待期间，其他 batch 释放了资源，所以在此尝试获取资源。
   3. 如果是 batch.size，并且释放的资源使得 free 空时，就从 free 中获取 byteBuffer 直接使用。
   4. 如果申请的资源正好是 BATCH.SIZE 的大小，并且释放的资源使得 bufferPool.free 不为空时，就从 free 中获取 byteBuffer 直接使用。否则释放掉一些 bufferPool.free 中的内存，直到够 size 用。
6. 移除正在等待内存资源的当前线程。通知其他等候内存资源的线程，有可用内存。
7. bufferPool 解锁，返回分配的 buffer。

```java
    public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");
        // 分配内存前对 bufferPool 加锁
        this.lock.lock();
        try {
            // check if we have a free buffer of the right size pooled
            // 如果申请的资源正好是 BATCH.SIZE 的大小，并且 bufferPool.free 链表中有可用的 ByteBuffer，直接使用
            if (size == poolableSize && !this.free.isEmpty())
                return this.free.pollFirst();

            // now check if the request is immediately satisfiable with the
            // memory on hand or if we need to block
            int freeListSize = this.free.size() * this.poolableSize;
            // bufferPool 可用内存大小 = this.nonPooledAvailableMemory + freeListSize，判断是否足够本次需要分配的 size
            if (this.availableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request
                // bufferPool 可用资源够用时，释放掉一些 bufferPool.free 中的 byteBuffer，确保够 size 使用
                freeUp(size);
                this.availableMemory -= size;
                // 解锁，返回分配的内存
                lock.unlock();
                return ByteBuffer.allocate(size);
            } else {
                // we are out of memory and will have to block
                // bufferPool 可用资源不够时，等待资源
                int accumulated = 0;
                ByteBuffer buffer = null;
                Condition moreMemory = this.lock.newCondition();
                long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                this.waiters.addLast(moreMemory);
                // loop over and over until we have a buffer or have reserved
                // enough memory to allocate one
                // 循环直到内存够用
                while (accumulated < size) {
                    long startWaitNs = time.nanoseconds();
                    long timeNs;
                    boolean waitingTimeElapsed;
                    try {
                        // 等待有内存资源释放
                        waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                    } catch (InterruptedException e) {
                        this.waiters.remove(moreMemory);
                        throw e;
                    } finally {
                        long endWaitNs = time.nanoseconds();
                        timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        this.waitTime.record(timeNs, time.milliseconds());
                    }

                    // 等待超时报错
                    if (waitingTimeElapsed) {
                        this.waiters.remove(moreMemory);
                        throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                    }

                    remainingTimeToBlockNs -= timeNs;
                    // check if we can satisfy this request from the free list,
                    // otherwise allocate memory
                    // 上面等待期间，其他 batch 释放了资源，所以在此尝试获取资源
                    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                        // just grab a buffer from the free list
                        // 如果申请的资源正好是 BATCH.SIZE 的大小，并且释放的资源使得 bufferPool.free 不为空时，就从 free 中获取 byteBuffer 直接使用
                        buffer = this.free.pollFirst();
                        accumulated = size;
                    } else {
                        // we'll need to allocate memory, but we may only get
                        // part of what we need on this iteration
                        // 释放掉一些 bufferPool.free 中的内存，直到够 size 用
                        freeUp(size - accumulated);
                        int got = (int) Math.min(size - accumulated, this.availableMemory);
                        this.availableMemory -= got;
                        accumulated += got;
                    }
                }

                // remove the condition for this thread to let the next thread
                // in line start getting memory
                // 移除正在等待内存资源的当前线程
                Condition removed = this.waiters.removeFirst();
                if (removed != moreMemory)
                    throw new IllegalStateException("Wrong condition: this shouldn't happen.");

                // signal any additional waiters if there is more memory left
                // over for them
                // 通知其他等候内存资源的线程，有可用内存
                if (this.availableMemory > 0 || !this.free.isEmpty()) {
                    if (!this.waiters.isEmpty())
                        this.waiters.peekFirst().signal();
                }

                // unlock and return the buffer
                // bufferPool 解锁，返回分配的 buffer
                lock.unlock();
                if (buffer == null)
                    return ByteBuffer.allocate(size);
                else
                    return buffer;
            }
        } finally {
            // 解锁
            if (lock.isHeldByCurrentThread())
                lock.unlock();
        }
    }
```

### BufferPool#deallocate -- 释放内存

org.apache.kafka.clients.producer.internals.BufferPool#deallocate(java.nio.ByteBuffer, int)

1. batch.size 大小的资源放到 byteBuffer.free 中。
2. 非 batch.size 大小的资源放到 availableMemory 中。

```java
	public void deallocate(ByteBuffer buffer, int size) {
        lock.lock();
        try {
            if (size == this.poolableSize && size == buffer.capacity()) {
                // batch.size 大小的资源直接放到 byteBuffer.free 中
                buffer.clear();
                this.free.add(buffer);
            } else {
                // 不是 batch.size 大小的资源放到 availableMemory 中
                this.availableMemory += size;
            }
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock();
        }
    }
```



## 个人思考

### RecordAccumulator 清理过期 batch 未自闭合

个人认为 RecordAccumulator 清理过期 batch 时未自闭合。

org.apache.kafka.clients.producer.internals.Sender#sendProducerData

1. 先由 sender 调用了 RecordAccumulator#expiredBatches 方法，过期 batch 出队。释放 batch 占用的内存靠调用处 (sender)自行调用另一个 api RecordAccumulator#deallocate。

2. 如果调用处(sender) 未调用 RecordAccumulator#deallocate 释放内存，可能导致内存泄漏。

3. 个人理解 RecordAccumulator 应该自闭合清理过期 batch 的操作，在一个 api 内保证过期 batch 出队时，调用自身 deallocate 方法释放内存，避免调用处疏忽导致内存泄漏。

```java
/**
     * 消息预发送, 把消息传递给 KafkaChanel 缓存
     */
    private long sendProducerData(long now) {
        // ...

        // create produce requests
        // 把按分区聚合的请求集合, 转换为按节点聚合的请求集合(因为网络 IO 是按节点发请求)
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
        addToInflightBatches(batches);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        accumulator.resetNextBatchExpiryTime();
        // 收集过期的 batch
        // Sender#inflightBatches 发送中的请求集合里过期的 batch
        List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
        // RecordAccumulator#batches 集合里过期的 batch
        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
        expiredBatches.addAll(expiredInflightBatches);

        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
        // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why
        // we need to reset the producer id here.
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
                + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
            // 处理过期 batch，内部调用 RecordAccumulator#deallocate 释放了过期 batch 占用的内存
            failBatch(expiredBatch, -1, NO_TIMESTAMP, new TimeoutException(errorMessage), false);
            if (transactionManager != null && expiredBatch.inRetry()) {
                // This ensures that no new batches are drained until the current in flight batches are fully resolved.
                transactionManager.markSequenceUnresolved(expiredBatch);
            }
        }
        sensors.updateProduceRequestMetrics(batches);

        // ...
        return pollTimeout;
    }
```



org.apache.kafka.clients.producer.internals.RecordAccumulator#expiredBatches

```java
    /**
     * Get a list of batches which have been sitting in the accumulator too long and need to be expired.
     */
    public List<ProducerBatch> expiredBatches(long now) {
        List<ProducerBatch> expiredBatches = new ArrayList<>();
        for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
            // expire the batches in the order of sending
            Deque<ProducerBatch> deque = entry.getValue();
            synchronized (deque) {
                while (!deque.isEmpty()) {
                    ProducerBatch batch = deque.getFirst();
                    if (batch.hasReachedDeliveryTimeout(deliveryTimeoutMs, now)) {
                        // 过期 batch 出队。释放 batch 占用的内存靠调用处自行调用另一个 api RecordAccumulator#deallocate
                        deque.poll();
                        batch.abortRecordAppends();
                        expiredBatches.add(batch);
                    } else {
                        maybeUpdateNextBatchExpiryTime(batch);
                        break;
                    }
                }
            }
        }
        return expiredBatches;
    }
```

