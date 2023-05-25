# kafka-consumer Rebalance 源码分析



## 说明

1. 本文基于 jdk 8, kafka 0.10.0 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## ConsumerCoordinator

在 consumer 中通过 ConsumerCoordinator 组件实现与服务端 GroupCoordinator 的交互，ConsumerCoordinator 继承了 AbstractCoordinator 抽象类。



![img](https://image-hosting.jellyfishmix.com/20230330220246)

图1处是收到正常的 JoinGroupResponse 响应回调，将 rejoinNeeded 设置为 false, 防止重复发送 JoinGroupRequest 请求。2, 3, 4 分别是收到异常的 SyncGroupResponse 或 HeartbeatResponse 或消费离开 ConsumerGroup 时执行的操作，这些情况会将 rejoinNeeded 设置为 true, 表示可以重新发送 JoinGroupRequest。



## AbstractCoordinator

```java
public abstract class AbstractCoordinator implements Closeable {

    private enum MemberState {
        UNJOINED,    // the client is not part of a group
        REBALANCING, // the client has begun rebalancing
        STABLE,      // the client has joined and is sending heartbeats
    }

    protected final int rebalanceTimeoutMs;
    private final int sessionTimeoutMs;
    private final boolean leaveGroupOnClose;
    private final GroupCoordinatorMetrics sensors;
    /**
     * 心跳任务的辅助类，其中记录了心跳相关的信息，例如: 两次发送心跳的间隔(interval)，最新发送心跳的时间(lastHeartbeatSend)，最后收到心跳响应的时间(lastHeartbeatReceive)等
     */
    private final Heartbeat heartbeat;
    /**
     * 当前消费者所属 ConsumerGroup 的 id
     */
    protected final String groupId;
    /**
     * ConsumerNetworkClient 对象，负责网络通信和执行定时任务
     */
    protected final ConsumerNetworkClient client;
    protected final Time time;
    protected final long retryBackoffMs;

    /**
     * 一个定时任务，负责定时发送心跳请求和心跳响应的处理，会被添加到 ConsumerNetworkClient.delayedTasks 定时任务队列中
     */
    private HeartbeatThread heartbeatThread = null;
    /**
     * 是否重新发送 JoinGroupRequest 的条件之一
     */
    private boolean rejoinNeeded = true;
    /**
     * 标记是否需要执行发送 JoinGroupRequest 请求前的准备操作
     */
    private boolean needsJoinPrepare = true;
    private MemberState state = MemberState.UNJOINED;
    private RequestFuture<ByteBuffer> joinFuture = null;
    /**
     * Node 类型，记录服务端 GroupCoordinator 所在的 Node 节点
     */
    private Node coordinator = null;
    private Generation generation = Generation.NO_GENERATION;
    
    protected static class Generation {
        /**
         * 服务端 GroupCoordinator 返回的年代信息，用来区分两次 rebalance 操作。由于网络延迟等问题，在执行Rebalance操作时可能受到上次Rebalance过程的请求，避免这种干扰，每次Rebalance操作都会递增generation的值。
         */
        public final int generationId;
        /**
         * 服务端 GroupCoordinator 返回的分配给消费者的唯一 id
         */
        public final String memberId;
        public final String protocol;
    }
    
    // ...
}
```



## addMetadataListener 方法，Metadata 更新的回调函数

```java
	private void addMetadataListener() {
        this.metadata.addListener(new Metadata.Listener() {
            @Override
            public void onMetadataUpdate(Cluster cluster, Set<String> unavailableTopics) {
                // if we encounter any unauthorized topics, raise an exception to the user
                if (!cluster.unauthorizedTopics().isEmpty())
                    throw new TopicAuthorizationException(new HashSet<>(cluster.unauthorizedTopics()));

                // AUTO_PATTERN 模式(按 pattern 订阅)的处理
                if (subscriptions.hasPatternSubscription())
                    updatePatternSubscription(cluster);

                // check if there are any changes to the metadata which should trigger a rebalance
                // 检测是否为 AUTO_PATTERN 或 AUTO_TOPICS 模式
                if (subscriptions.partitionsAutoAssigned()) {
                    // 创建 MetadataSnapshot 快照并记录
                    MetadataSnapshot snapshot = new MetadataSnapshot(subscriptions, cluster);
                    if (!snapshot.equals(metadataSnapshot))
                        metadataSnapshot = snapshot;
                }

                if (!Collections.disjoint(metadata.topics(), unavailableTopics))
                    // 更新 needUpdate 字段为 true，表示需要重新更新 metadata
                    metadata.requestUpdate();
            }
        });
    }
```

调用 subscribe() 方法之后，consumer 所做的事情，分两种情况，一种按 topic 列表订阅，一种是按 pattern 模式订阅：

1. topic 列表订阅:
   1. 更新 SubscriptionState 中记录的 subscription(记录的是订阅的 topic 列表)，将 SubscriptionType 类型设置为 AUTO_TOPICS。
   2. 更新 metadata 中的 topic 列表(topics 变量)，并请求更新 metadata。
2. pattern 模式订阅:
   1. 更新 SubscriptionState 中记录的 subscribedPattern，设置为 pattern，将 SubscriptionType 类型设置为 AUTO_PATTERN。
   2. 设置 Metadata 的 needMetadataForAllTopics 为 true，即在请求 metadata 时，需要更新所有 topic 的 metadata 信息，设置后再请求更新 metadata。
   3. 调用 coordinator.updatePatternSubscription() 方法，遍历所有 topic 的 metadata，找到所有满足 pattern 的 topic 列表，更新到 SubscriptionState 的 subscriptions 和 Metadata 的 topics 中。
   4. 通过在 ConsumerCoordinator 中调用 addMetadataListener() 方法在 Metadata 中添加 listener。当每次触发 metadataUpdate 时就调用 coordinator.updatePatternSubscription() 方法更新，如果本地缓存的 topic 列表与现在要订阅的 topic 列表不同，触发 rebalance 操作。

其他部分，两者基本一样，只是 pattern 模型在每次更新 topic-metadata 时，获取全局的 topic 列表，如果发现有新加入的符合条件的 topic，就立马去订阅，其他的地方，包括 Group 管理、topic-partition 的分配都是一样的。



## PartitionAssignor

leaderConsumer 收到 JoinGroupResponse 后，会按照指定的分区分配策略进行分区分配，分区分配策略是 PartitionAssignor 接口的实现。

![img](https://image-hosting.jellyfishmix.com/20230406200453)

![img](https://image-hosting.jellyfishmix.com/20230406203951)





1. PartitionAssignor 接口中定义了 Assignment 和 Subscription 两个内部类。声明了分区分配需要的两方面数据：Metadata 中记录的集群元数据 Assignment 和每个 consumer 的订阅信息 Subscription。
2. 为了增强 consumer 用户对分配策略的控制，将用户订阅信息和影响分配结果的用户自定义信息封装在 Subscription 中，topics 集合标识某 consumer 订阅的 Topic 集合，userData 表示用户自定义的数据，可以是每个 consumer 的权重。
3. PartitionAssignor 接口提供了 subscription() 方法，用于添加用户自定义的数据 Subscription#userData，在创建 JoinGroupRequest 的时候会用到 subscription() 方法。
4. Assignment 中保存了分区的分配结果，partitions 表示分配给某消费者的 TopicPartition 集合，userData 是用户自定义的数据。
5. 再看一下 PartitionAssignor 接口的其他方法，assign() 是完成 Partition 分配的抽象方法。onAssignment() 方法是在每个消费者收到 leader 分配结果时的回调函数，此调用发生在 SyncGroupResponse 之后。



## AbstractPartitionAssignor

## assign 分区分配方法

1. AbstractPartitionAssignor 是 PartitionAssignor 接口的实现。
2. 分配过程中不使用用户自定义数据 userData，如果分区分配策略需要 userData，不能直接继承 AbstractPartitionAssignor, 需要直接实现 PartitionAssignor。

具体逻辑:

1. 解析 Subscriptions 集合，获取 topic 信息，丢弃了 userData 信息。
2. 可以看出默认分区分配策略并不使用 userData。如果分区分配策略需要 userData，不能直接继承 AbstractPartitionAssignor, 需要直接实现 PartitionAssignor。
3. 统计每个 Topic 的分区个数。
4. 具体分区分配策略，模版方法，由子类实现。
5. 整理分区分配结果。

```java
    /**
     * assign 分区分配方法，AbstractPartitionAssignor 是 PartitionAssignor 接口的实现
     * 分配过程中不使用用户自定义数据 userData，如果分区分配策略需要 userData，不能直接继承 AbstractPartitionAssignor, 需要直接实现 PartitionAssignor
     */
    @Override
    public Map<String, Assignment> assign(Cluster metadata, Map<String, Subscription> subscriptions) {
        Set<String> allSubscribedTopics = new HashSet<>();
        Map<String, List<String>> topicSubscriptions = new HashMap<>();
        // 解析 Subscriptions 集合，获取 topic 信息，丢弃了 userData 信息。
        // 可以看出默认分区分配策略并不使用 userData。如果分区分配策略需要 userData，不能直接继承 AbstractPartitionAssignor, 需要直接实现 PartitionAssignor
        for (Map.Entry<String, Subscription> subscriptionEntry : subscriptions.entrySet()) {
            List<String> topics = subscriptionEntry.getValue().topics();
            allSubscribedTopics.addAll(topics);
            topicSubscriptions.put(subscriptionEntry.getKey(), topics);
        }

        // 统计每个 Topic 的分区个数
        Map<String, Integer> partitionsPerTopic = new HashMap<>();
        for (String topic : allSubscribedTopics) {
            Integer numPartitions = metadata.partitionCountForTopic(topic);
            if (numPartitions != null && numPartitions > 0)
                partitionsPerTopic.put(topic, numPartitions);
            else
                log.debug("Skipping assignment for topic {} since no metadata is available", topic);
        }

        // 具体分区分配策略，模版方法，由子类实现
        Map<String, List<TopicPartition>> rawAssignments = assign(partitionsPerTopic, topicSubscriptions);

        // this class has maintains no user data, so just wrap the results
        // 整理分区分配结果
        Map<String, Assignment> assignments = new HashMap<>();
        for (Map.Entry<String, List<TopicPartition>> assignmentEntry : rawAssignments.entrySet())
            assignments.put(assignmentEntry.getKey(), new Assignment(assignmentEntry.getValue()));
        return assignments;
    }
```



## AbstractPartitionAssignor 的预设子类 RangeAssignor 和 RoundRobinAssignor

AbstractPartitionAssignor 的预设子类 RangeAssignor 和 RoundRobinAssignor，实现了具体的两种分区分配策略。

- RangeAssignor: 针对每个 topic, n = 分区数 / consumer 数量，m=分区数 % consumer 数量，前 m 个消费者每个分配 n+1 个分区，后面的 (消费者数量-m) 个消费者每个分配 n 个分区。
- RoundRobinAssignor 实现原理是: 将所有 Topic的Partition 按照字典排列，对每个 consumer 进行轮询分配。