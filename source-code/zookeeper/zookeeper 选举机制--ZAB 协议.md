# zookeeper 选举机制--ZAB 协议



## 说明

1. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## zookeeper 节点角色

### leader

1. leader 定位是分布式系统中的 master。leader 提供读写能力，且写请求只能由 leader 来完成。
2. 每一次写请求都会生成一个 zxid。然后将此请求同步给各个 follower 和 observer，zxid 也是决定顺序一致性的关键所在。

### follower

1. follower 是分布式系统中常见的从节点。只提供读请求的能力，如果写请求打到 follower 上了，那么 follower 会将此请求转发到 leader，由 leader 来完成写入，再同步给 follower。
2. follower 不只提供读能力，还有选举的能力。如果 leader 挂了的话，follower 是有资格选举并成为新 leader 的。

### observer

1. observer 可以直接提升我们 zookeeper 集群的并发读能力。
2. observer 只提供读能力，和 follower 的区别在于 observer 没有选举并成为 leader 的能力。



## myid

搭建 zookeeper 集群时，配置中指定的server.1的.1 .2 .3为 myid

```properties
server.1=ZooKeeper1:2888:3888
server.2=ZooKeeper2:2888:3888
server.3=ZooKeeper3:2888:3888
```



## zxid

1. zookeeper 每个节点都维护了一个 zxid
2. 高 32 位为 epoch 朝代, 低 32 位为事务计数(写请求计数)。
3. 每次写请求，事务计数都会+1, 事务计数写满后 epoch 朝代会+1

```
00000000000000000000000000000000 00000000000000000000000000000000
```



## leader 选举机制-投票比较大小

1. 比较 epoch，epoch 大者胜出。均相等则继续比较事物计数。
2. 比较事物计数，事务计数大者胜出，均相等则继续比较 myid。
3. 比较 myid，myid 大者胜出。

![持久化-4.png](https://image-hosting.jellyfishmix.com/20231107014430.png)



## leader 选举机制--选举状态

1. LOOKING: 竞选状态，也就是说此状态下还没有 Leader 诞生，需要进行 Leader 选举。

2. FOLLOWING: Follower 状态，对应我们之前介绍的 Follower 角色对应的状态，并且它自身是知道 Leader 是谁的。

3. OBSERVING: Observer 状态，对应我们之前介绍的 Observer 角色对应的状态，并且它自身是知道 Leader 是谁的。

4. LEADING: Leader 状态，对应我们之前介绍的 Leader 角色对应的状态。



## leader 选举机制--投票过程

### 第一种情况: 集群刚启动时，谁来当 Leader

假设整个集群有两个节点，ZooKeeper A 和 ZooKeeper B。

1. ZooKeeper A 先投票给自己，投票信息包含节点 sid 和 zxid，sid 就是 myid ，是在配置文件当中配置死的，整个集群内唯一，zxid 是事务 id，自增。假设 ZooKeeper A 的信息为 (1,0)。

2. ZooKeeper B 也投票给自己，假设 ZooKeeper B 的 sid 为 2 ，那么此时 ZooKeeper B 投票信息为 (2,0)。

3. 接下来 ZooKeeper A 和 ZooKeeper B 分别将自己的投票信息广播给集群中其他节点。说白了就是 ZooKeeper A 将（1,0） 广播给 ZooKeeper B， ZooKeeper B 将（2,0）广播给 ZooKeeper A。

4. ZooKeeper A 收到 ZooKeeper B 的投票信息后，检查下 ZooKeeper B 的状态是否是本轮投票，以及是否是 LOOKING 寻主的状态。 反之，ZooKeeper B 收到 ZooKeeper A 的投票信息后也是一样的。

5. 然后进行投票对比：ZooKeeper A 会将自己的投票和 ZooKeeper B 的投票进行对比，优先对比 zxid，其次对比 sid。

   1. 如果 ZooKeeper B 的 zxid 较大，则把自己的投票信息更新为 ZooKeeper B 的投票信息。

   2. 如果 zxid 相等，则比较 sid，sid 大的胜出。这里 ZooKeeper A 的 sid 是 1，ZooKeeper B 的 sid 是 2，所以 ZooKeeper A 更新投票信息为 (2，0)，然后将投票信息再次发送出去。而 ZooKeeper B 不需要更新投票信息，但是下一轮还需要再次将投票发出去。

6. 统计投票：每一轮投票都会统计每台节点收到的投票信息，判断是否有过半的节点收到了相同的投票信息。ZooKeeper A 和 ZooKeeper B 收到的投票信息都为 (2，0)，且数量来说，大于一半节点的数量，所以将 ZooKeeper B 选出来作为 Leader。

7. 最后更新节点状态：ZooKeeper A 作为 Follower，更新状态为 FOLLOWING，ZooKeeper B 作为 Leader，更新状态为 LEADING。

### 第二种情况: 老 leader 崩溃，重新选举 leader

重复上述投票过程，选举出新 leader 后，如老 leader 重新启动，无权继续当 leader，此时集群中已经有了新 leader，崩溃的老 leader 重新启动后成为 follower。

