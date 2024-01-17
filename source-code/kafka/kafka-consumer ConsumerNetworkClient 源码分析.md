# kafka-consumer ConsumerNetworkClient 源码分析



## 说明

1. 本文基于 kafka 0.10.0 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## ConsumerNetworkClient 属性

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient

kafka consumer 负责网络通信的组件。

```java
public class ConsumerNetworkClient implements Closeable {
    private static final Logger log = LoggerFactory.getLogger(ConsumerNetworkClient.class);

    /**
     * NetworkClient 对象
     */
    private final KafkaClient client;
    /**
     * 由调用 KafkaConsumer 对象的消费者线程之外的其他线程设置，表示要中断 KafkaConsumer 线程
     */
    private final AtomicBoolean wakeup = new AtomicBoolean(false);
    /**
     * 心跳任务的定时任务队列，DelayedTaskQueue 是 Kafka 提供的定时任务队列的实现，其底层是使用 JDK 提供的 PriorityQueue 实现。
     * PriorityQueue 是一个非线程安全的，无界的，优先级队列，实现原理是小顶堆，底层基于数组实现，对应的线程安全版实现是 PriorityBlockingQueue。
     */
    private final DelayedTaskQueue delayedTasks = new DelayedTaskQueue();
    /**
     * 缓冲队列，key: node节点，value: 发往此 node 的 ClientRequest 集合
     */
    private final Map<Node, List<ClientRequest>> unsent = new HashMap<>();
    /**
     * 用于管理 kafka 集群元数据
     */
    private final Metadata metadata;
    private final Time time;
    private final long retryBackoffMs;
    /**
     * ClientRequest 在 unsent 中缓存的超时时长配置
     */
    private final long unsentExpiryMs;

    // this count is only accessed from the consumer's main thread
    /**
     * kafkaConsumer 是否正在执行不可中断的方法。每进入一个不可中断的方法则 + 1，退出不可中断方法时，则 - 1。
     * 只会被当前使用 kafkaConsumer 的 thread 修改，其他线程不能修改。
     */
    private int wakeupDisabledCount = 0;
    
    // ...
}
```



## ConsumerNetworkClient#send

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient#send

将待发送的请求封装成 ClientRequest，然后保存在 unsent 集合中等待发送。ConsumerNetworkClient#poll 方法中会真正地发送 ClientRequest 请求。

```java
	public RequestFuture<ClientResponse> send(Node node,
                                              ApiKeys api,
                                              AbstractRequest request) {
        long now = time.milliseconds();
        RequestFutureCompletionHandler future = new RequestFutureCompletionHandler();
        RequestHeader header = client.nextRequestHeader(api);
        RequestSend send = new RequestSend(node.idString(), header, request.toStruct());
        // 创建 ClientRequest 对象，保存到 unsent 集合中
        put(node, new ClientRequest(now, true, send, future));
        return future;
    }
```



## ConsumerNetworkClient#poll 方法

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient#poll(long, long, boolean)

poll 方法是 ConsumerNetworkClient 的核心方法，poll 方法有多个不同入参的封装，最终会调用 poll(long timeout, long now, boolean executeDelayedTasks) 全参方法重载，入参数的含义:

1. timeout 表示 poll 方法最长的阻塞时间。
2. now 表示当前的时间戳。
3. executeDelayedTasks 表示是否执行 delayedTasks 队列中的定时任务。

需要注意的是: 发送 ClientRequest 分成两步操作，先执行 trySend 再执行 NetworkClient#poll。trySend 先把所有准备好的 ClientRequest 都设置到对应的 channel, 然后 NetworkClient#poll 把所有写事件准备就绪的 channel 找出来，执行写操作。

具体逻辑:

1. 调用 trySend 方法，处理 unsent 中缓存的请求，尝试发送请求。
2. 计算超时时间，由 timeout 和 delayedTasks 队列中最近要执行的定时任务时间，共同决定。
3. 调用 clientPoll 方法，将 KafkaChannel#send 字段指定的消息发送出去。clientPoll 方法可能会更新 Metadata, 并且使用一系列 handle 方法处理请求的响应，连接断开，超时等情况，并调用请求的回调函数。
4. 上面 clientPoll 方法可能更新了 metadata，这里检测 consumer 和 node 的连接状态，如果连接断开则移除 unsent 中 node 对应的 ClientRequest 并调用回调函数。
5. 根据 executeDelayedTasks 参数决定是否处理 delayedTasks 队列中的定时任务。
6. 再次调用 trySend 方法。前面调用了 NetworkClient.poll 方法，在其中可能已经将 KafkaChannel.send 字段上的请求发送出去了。也可能新建了与某些 node 的连接，所以再次调用 trySend 方法给 node 的 channel 设置 ClientRequest。
7. ClientRequest 超时回调处理。

```java
private void poll(long timeout, long now, boolean executeDelayedTasks) {
        // send all the requests we can send now
        // 处理 unsent 中缓存的请求，尝试发送请求
        trySend(now);

        // ensure we don't poll any longer than the deadline for
        // the next scheduled task
        // 计算超时时间，由 timeout 和 delayedTasks 队列中最近要执行的定时任务时间，共同决定
        timeout = Math.min(timeout, delayedTasks.nextTimeout(now));
        // 调用 clientPoll 方法，将 KafkaChannel#send 字段指定的消息发送出去
        // clientPoll 方法可能会更新 Metadata, 并且使用一系列 handle 方法处理请求的响应，连接断开，超时等情况，并调用请求的回调函数
        clientPoll(timeout, now);
        now = time.milliseconds();

        // handle any disconnects by failing the active requests. note that disconnects must
        // be checked immediately following poll since any subsequent call to client.ready()
        // will reset the disconnect status
        // 上面 clientPoll 方法可能更新了 metadata，这里检测 consumer 和 node 的连接状态，如果连接断开则移除 unsent 中 node 对应的 ClientRequest 并调用回调函数
        checkDisconnects(now);

        // execute scheduled tasks
        // 根据 executeDelayedTasks 参数决定是否处理 delayedTasks 队列中的定时任务
        if (executeDelayedTasks)
            delayedTasks.poll(now);

        // try again to send requests since buffer space may have been
        // cleared or a connect finished in the poll
        // 再次调用 trySend 方法。前面调用了 NetworkClient.poll 方法，在其中可能已经将 KafkaChannel.send 字段上的请求发送出去了。
        // 也可能新建了与某些 node 的连接，所以再次调用 trySend 方法给 node 的 channel 设置 ClientRequest。
        trySend(now);

        // fail requests that couldn't be sent if they have expired
        // ClientRequest 超时回调处理
        failExpiredRequests(now);
    }
```

### ConsumerNetworkClient#trySend 方法

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient#trySend

处理 unsent 中缓存的请求，尝试发送请求

```java
	private boolean trySend(long now) {
        // send any requests that can be sent now
        boolean requestsSent = false;
        // 遍历每个 node 节点的 unsent 集合
        for (Map.Entry<Node, List<ClientRequest>> requestEntry: unsent.entrySet()) {
            Node node = requestEntry.getKey();
            // 遍历此 node 所有的 ClientRequest
            Iterator<ClientRequest> iterator = requestEntry.getValue().iterator();
            while (iterator.hasNext()) {
                ClientRequest request = iterator.next();
                // 调用 NetworkClient#ready 方法，检测消费者与此节点之间的连接是否可以发送请求。
                // 如果一个 node 的上一个 ClientRequest 还没有发送，那么当前 ClientRequest 无法发送。
                if (client.ready(node, now)) {
                    // 将请求放入 InFlightRequests 队列中，然后放入 KafkaChannel#send 字段中等待发送
                    client.send(request, now);
                    // 从 unsent 集合中删除此请求
                    iterator.remove();
                    requestsSent = true;
                }
            }
        }
        return requestsSent;
    }
```

### NetworkClient#poll -- 真正把 ClientRequest 发送到网络中

org.apache.kafka.clients.NetworkClient#poll

socket 相关网络 IO，真正把 ClientRequest 发送到网络中。

1. 在前面的 trySend 方法中，为 node 对应的通道注册了写事件。poll 方法中写事件准备就绪，处理通道的写操作将数据写到网络中。
2. trySend 方法可能设置为多个 node 的 channel 设置请求，所以会有多个 channel 的写事件准备就绪，因此 poll 方法中会发送多个请求。

具体逻辑:

1. 判断是否需要更新 metadata 元数据。
2. 调用 Selector.poll() 进行 socket 相关的 IO 操作。
3. 使用一系列 handle 方法处理 response。

```java
	@Override
    public List<ClientResponse> poll(long timeout, long now) {
        // 判断是否需要更新 metadata 元数据
        long metadataTimeout = metadataUpdater.maybeUpdate(now);
        try {
            // 调用 Selector.poll() 进行 socket 相关的 IO 操作
            this.selector.poll(Utils.min(timeout, metadataTimeout, requestTimeoutMs));
        } catch (IOException e) {
            log.error("Unexpected error during I/O", e);
        }

        // process completed actions
        // 使用一系列 handle 方法处理 response
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleTimedOutRequests(responses, updatedNow);

        // invoke callbacks
        for (ClientResponse response : responses) {
            if (response.request().hasCallback()) {
                try {
                    response.request().callback().onComplete(response);
                } catch (Exception e) {
                    log.error("Uncaught error in request completion:", e);
                }
            }
        }

        return responses;
    }
```

### ConsumerNetworkClient#checkDisconnects 方法 -- 检测 consumer 和 node 的连接状态

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient#checkDisconnects

检测 consumer 和 node 的连接状态，如果连接失败则移除 unsent 中 node 对应的 ClientRequest 并调用回调函数。

1. 遍历 unsent 集合中的每个 node，检测 consumer 与每个 node 的连接状态是否失败。
2. 如果连接状态失败，从 unsent 集合中移除此 node 对应的全部 ClientRequest，调用被移除的 ClientRequest 的回调函数。

```java
	private void checkDisconnects(long now) {
        // any disconnects affecting requests that have already been transmitted will be handled
        // by NetworkClient, so we just need to check whether connections for any of the unsent
        // requests have been disconnected; if they have, then we complete the corresponding future
        // and set the disconnect flag in the ClientResponse
        // 遍历 unsent 集合中的每个 node
        Iterator<Map.Entry<Node, List<ClientRequest>>> iterator = unsent.entrySet().iterator();
        while (iterator.hasNext()) {
            Map.Entry<Node, List<ClientRequest>> requestEntry = iterator.next();
            Node node = requestEntry.getKey();
            // 检测 consumer 与每个 node 的连接状态是否失败
            if (client.connectionFailed(node)) {
                // Remove entry before invoking request callback to avoid callbacks handling
                // coordinator failures traversing the unsent list again.
                // 从 unsent 集合中移除此 node 对应的全部 ClientRequest
                iterator.remove();
                // 调用被移除的 ClientRequest 的回调函数
                for (ClientRequest request : requestEntry.getValue()) {
                    RequestFutureCompletionHandler handler =
                            (RequestFutureCompletionHandler) request.callback();
                    handler.onComplete(new ClientResponse(request, now, true, null));
                }
            }
        }
    }
```

### ConsumerNetworkClient#failExpiredRequests 方法 -- ClientRequest 超时回调处理

org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient#failExpiredRequests

遍历 unsent 集合，检测每个 ClientRequest 是否超时，调用超时 ClientRequest 的回调函数，并将其从 unsent 集合删除。

```java
	private void failExpiredRequests(long now) {
        // clear all expired unsent requests and fail their corresponding futures
        Iterator<Map.Entry<Node, List<ClientRequest>>> iterator = unsent.entrySet().iterator();
        // 遍历 unsent 集合，检测每个 ClientRequest 是否超时，调用超时 ClientRequest 的回调函数，并将其从 unsent 集合删除
        while (iterator.hasNext()) {
            Map.Entry<Node, List<ClientRequest>> requestEntry = iterator.next();
            Iterator<ClientRequest> requestIterator = requestEntry.getValue().iterator();
            while (requestIterator.hasNext()) {
                ClientRequest request = requestIterator.next();
                // 检测当前 ClientRequest 是否超时
                if (request.createdTimeMs() < now - unsentExpiryMs) {
                    // 调用超时 ClientRequest 的回调函数，并将其从 unsent 集合删除
                    RequestFutureCompletionHandler handler =
                            (RequestFutureCompletionHandler) request.callback();
                    handler.raise(new TimeoutException("Failed to send request after " + unsentExpiryMs + " ms."));
                    requestIterator.remove();
                } else
                    break;
            }
            if (requestEntry.getValue().isEmpty())
                iterator.remove();
        }
    }
```

