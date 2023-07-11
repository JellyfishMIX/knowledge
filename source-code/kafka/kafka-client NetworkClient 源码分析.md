# kafka-client NetworkClient 源码分析



## 说明

1. 本文基于 kafka 2.7 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## NetworkClient 的属性

```java
public class NetworkClient implements KafkaClient {

    private enum State {
        ACTIVE,
        CLOSING,
        CLOSED
    }

    /* the selector used to perform network i/o */
    // 用于网络 IO 的选择器
    private final Selectable selector;

    /**
     * 用于元数据的更新
     */
    private final MetadataUpdater metadataUpdater;

    private final Random randOffset;

    /* the state of each node's connection */
    /**
     * 集群所有连接的状态都在这里管理
     */
    private final ClusterConnectionStates connectionStates;

    /* the set of requests currently being sent or awaiting a response */
    /**
     * 发送后还没有响应的请求集合
     */
    private final InFlightRequests inFlightRequests;

    /* the socket send buffer size in bytes */
    /**
     * 表示发送数据的缓冲区的大小
     */
    private final int socketSendBuffer;

    /* the socket receive size buffer in bytes */
    /**
     * 表示接收数据的缓冲区的大小
     */
    private final int socketReceiveBuffer;

    /* the client id used to identify this client in requests to the server */
    /**
     * client 的 id
     */
    private final String clientId;

    /* the current correlation id to use when sending requests to servers */
    private int correlation;

    /* default timeout for individual requests to await acknowledgement from servers */
    private final int defaultRequestTimeoutMs;

    /* time in ms to wait before retrying to create connection to a server */
    /**
     * 重连的退避时间
     */
    private final long reconnectBackoffMs;

    private final ClientDnsLookup clientDnsLookup;

    private final Time time;

    /**
     * True if we should send an ApiVersionRequest when first connecting to a broker.
     *
     * true: 当第一次连接一个 broker 的时候, 我们应当发送一个 version 的请求, 用来得知 broker 的版本 false: 不发 version 的请求
     */
    private final boolean discoverBrokerVersions;

    /**
     * broker 的版本
     */
    private final ApiVersions apiVersions;

    /**
     * key: nodeId
     */
    private final Map<String, ApiVersionsRequest.Builder> nodesNeedingApiVersionsFetch = new HashMap<>();

    /**
     * 取消的请求
     */
    private final List<ClientResponse> abortedSends = new LinkedList<>();
    
    // ...
```

### InFlightRequests -- 用来存储和操作待发送消息的缓存区

两个关键字段:

1. 单个 node 连接持有请求数上限，默认 5。
2. key: nodeId, value: 此 node 对应的发送中请求。

```java
final class InFlightRequests {

    /**
     * 单个 node 连接持有请求数上限，默认 5
     */
    private final int maxInFlightRequestsPerConnection;
    /**
     * key: nodeId, value: 此 node 对应的发送中请求
     */
    private final Map<String, Deque<NetworkClient.InFlightRequest>> requests = new HashMap<>();
}
```

存储结构如下:

InFlightRequests#requests 是个map集合，每个 node 对应一个请求队列。每次请求进入 InFlightRequests前，会先判断请求属于哪个node，然后加入对应的队列。

![D9C0A00A-4335-4457-96F4-00E35CFCA78B.png](https://image-hosting.jellyfishmix.com/20230531231911.png)

#### InFlightRequests 乱序问题

如果某个请求超时，同时配置的重试次数大于 0，就会发生重试。且重试的请求会重新排在队列的后面，与最初的发送顺序不一致，造成了乱序。

解决方案:

1. 重试次数设置为 0，这样即使请求超时也不会重试导致乱序。弊端: 这种方式可能会丢失消息。
2. 把 maxInFlightRequestsPerConnection 设置为1，这样上一个请求发送成功时，下一个请求才会开始发送。弊端: 请求吞吐量和发送性能降低。



## NetworkClient#send 方法 -- 准备发送

org.apache.kafka.clients.NetworkClient#send

这里的发送数据是把数据发送到缓存里，并不是真正的网络发送。为下一步真正的网络 IO 服务。

1. 把要发送的请求封装成 inFlightRequest，放到对应 channel 的字段 NetworkSend 里缓存起来。
2. NetworkSend 是对 NIOBuffer 的封装。

```java
    @Override
    public void send(ClientRequest request, long now) {
        doSend(request, false, now);
    }

    private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now) {
        ensureActive();
        String nodeId = clientRequest.destination();
        if (!isInternalRequest) {
            /*
             * 此 broker node 是否可接收请求
             * 1. 连接是否正常
             * 2. channel 连接是否建立
             * 3. inFlightRequests.canSendMore(node) 是否还能接收请求
             */
            if (!canSendRequest(nodeId, now))
                throw new IllegalStateException("Attempt to send a request to node " + nodeId + " which is not ready.");
            // ...
            doSend(clientRequest, isInternalRequest, now, builder.build(version));
        } catch (UnsupportedVersionException unsupportedVersionException) {
            // ...
        }
    }

    private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {
        String destination = clientRequest.destination();
        RequestHeader header = clientRequest.makeHeader(request.version());
        if (log.isDebugEnabled()) {
            log.debug("Sending {} request with header {} and timeout {} to node {}: {}",
                clientRequest.apiKey(), header, clientRequest.requestTimeoutMs(), destination, request);
        }
        // 构建 NetworkSend 对象
        Send send = request.toSend(destination, header);
        // 构建 inFlightRequest 对象
        InFlightRequest inFlightRequest = new InFlightRequest(
                clientRequest,
                header,
                isInternalRequest,
                request,
                send,
                now);
        // 加入 inFlightRequest 的集合中
        this.inFlightRequests.add(inFlightRequest);
        selector.send(send);
    }
```



## NetworkClient#poll 方法 -- 真正把 ClientRequest 发送到网络中

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
        ensureActive();

        if (!abortedSends.isEmpty()) {
            // If there are aborted sends because of unsupported version exceptions or disconnects,
            // handle them immediately without waiting for Selector#poll.
            List<ClientResponse> responses = new ArrayList<>();
            handleAbortedSends(responses);
            completeResponses(responses);
            return responses;
        }

        // 判断并尝试更新元数据
        long metadataTimeout = metadataUpdater.maybeUpdate(now);
        try {
            // 调用 Selector.poll() 进行 socket 相关的 IO 操作
            this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
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
        handleInitiateApiVersionRequests(updatedNow);
        handleTimedOutConnections(responses, updatedNow);
        handleTimedOutRequests(responses, updatedNow);
        completeResponses(responses);

        return responses;
    }
```

