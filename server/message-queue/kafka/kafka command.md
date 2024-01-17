# kafka command



## zookeeper

### 安装

[在Linux上安装Zookeeper - 二木成林 - CSDN](https://blog.csdn.net/cnds123321/article/details/123621681)

可能会用到的命令：

#### 利用 curl 下载文件

使用内置 option：-o(小写)，自行指定下载后文件名称

```bash
curl -o zookeeper-3.8.0.tar.gz https://dlcdn.apache.org/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz
```

使用内置 option：-O(大写)，以下载来源服务器上的名称保存文件到本地

```bash
curl -O https://dlcdn.apache.org/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz
```

#### 利用 tar 解压

```bash
tar -zxvf ./xxx -C /xxx/
# e.g.
tar -zxvf /root/home/apache-zookeeper-3.8.0-bin.tar.gz -C /root/programming/server
```

tar是用来建立，还原备份文件的工具程序，它可以加入，解开备份文件内的文件。

- `-z` 通过gzip指令处理备份文件。
- `-x` 从备份文件中还原文件。
- `-v` 显示指令执行过程。
- `-f` 指定备份文件。
- `-C` <目的目录> 切换到指定目录。

#### 文件或目录重命名 mv

```bash
mv apache-zookeeper-3.8.0-bin zookeeper-3.8.0
```

### 属性信息

version=3.8.0

default port=2181

安装根目录

```bash
/root/Programming/Server/zookeeper-3.8.0
```

dataPath

```bash
/var/data/zookeeper
```

confgPath

```bash
/root/Programming/Server/zookeeper-3.8.0/conf/zoo.cfg
```

### 命令

启动 zookeeper 服务端

```bash
cd /root/Programming/Server/zookeeper-3.8.0/bin
./zkServer.sh start
```

查看 zookeeper 状态

```bash
cd /root/Programming/Server/zookeeper-3.8.0/bin
./zkServer.sh status
```

停止 zookeeper 服务端

```bash
cd /root/Programming/Server/zookeeper-3.8.0/bin
./zkServer.sh stop
```

启动 zookeeper 客户端

```bash
cd /root/Programming/Server/zookeeper-3.8.0/bin
./zkCli.sh
```

在 zookeeper 客户端中输入，查看 zk 的根目录

```bash
ls /
```

在 zookeeper 客户端中输入，查看节点信息

```
get
```

在 zookeeper 客户端中输入，退出 zookeeper 客户端

```bash
quit
```

安装配置参考文档：[在Linux上安装Zookeeper - 二木成林 - CSDN](https://blog.csdn.net/cnds123321/article/details/123621681)

### 可能会遇到的问题

#### zookeeper 启动后可能会遇到的情况，有额外的端口占用

使用 `netstat -ntlp` 命令发现，除了 zookeeper 占用的 2182 端口以外还有一个随机端口和8080端口启用了。如需查看原因或关闭请见：[zookeeper关闭额外随机端口 - SangBigYe - CSDN](https://blog.csdn.net/CutelittleBo/article/details/123275385)



## kafka

### kafka 安装

[Linux 安装kafka详细步骤 - 不像程序猿的程序员 - CSDN](https://blog.csdn.net/qq_44538738/article/details/114005478)

### 属性信息

version=2.5.1(kafka 从 2.8 开始移除对外部 kafka 的依赖，我们先使用 2.8 之前的版本，比较经典，模型机制容易理解。)

default port=9092

安装根目录

```bash
/root/Programming/Server/kafka-2.5.1
```

confgPath

```bash
/root/Programming/Server/kafka-2.5.1/config/server.properties
```

logPath

```bash
/var/log/kafka
```

### 命令

#### 启动 kafka 服务端 broker

```bash
cd /root/Programming/Server/kafka-2.5.1/bin
nohup ./kafka-server-start.sh ../config/server.properties > /dev/null 2>&1 &
```

#### 停止 kafka 服务端 broker

```bash
cd /root/Programming/Server/kafka-2.5.1/bin
./kafka-server-stop.sh
```

安装配置参考文档：[Linux 安装kafka详细步骤 - 不像程序猿的程序员 - CSDN](https://blog.csdn.net/qq_44538738/article/details/114005478)

### 可能会遇到的问题

#### kafka broker 启动失败，进程意外退出，没有成功运行

请暂时使用以下命令启动 kafka 服务端 broker，查看报错信息

```bash
cd /root/Programming/Server/kafka-2.5.1/bin
./kafka-server-start.sh ../config/server.properties
```

#### kafka broker 启动时报错 error='Not enough space'

[kafka使用时的问题 - codeg - 博客园](https://www.cnblogs.com/darklights/p/13414279.html)

#### kafka broker 启动后可能会遇到的情况，有额外的端口占用

使用 `netstat -ntlp` 命令发现，kafka 又开了两个额外的端口，无影响，不用管。如需查看原因或关闭请见：[KAFKA随机产生JMX 端口指定的问题 - Suntoma - CSDN](https://blog.csdn.net/weixin_40209426/article/details/82217987)

#### kafka client 项目启动时报错 Connection to node -1 could not be established. Broker may not be available.

参考：[SpringBoot解决启动kafka：Connection to node -1 could not be established. Broker may not be available - 野花太放肆 - CSDN](https://blog.csdn.net/yu980219/article/details/112301399)

kafka broker 没有开启外网 ip 访问，需要配置。

1. listeners 指明 kafka 当前节点监听什么来源的 ip
2. advertised.listeners 指明客户端通过哪个 ip 可以访问到当前节点

kafkaPath/conf/server.properties 修改：

```properties
# listeners=PLAINTEXT://XXXX:9092
listeners=PLAINTEXT://0.0.0.0:9092
# advertised.listeners=PLAINTEXT://your.host.name:9092，your.host.name 改成域名或外网 ip
advertised.listeners=PLAINTEXT://your.host.name:9092
```

如果 kafka broker 运行在服务器上，最关键的地方是：

```properties
# 允许外部端口连接
listeners=PLAINTEXT://0.0.0.0:9092
```

这里不能设置成 localhost，必须设置为 0.0.0.0 允许所有 ip 的访问，否则只有 localhost 服务器自己才能访问，服务器外部无法访问。

#### kafka broker 启动时报错 java.net.bindException: cannot assign requested address

参考：

1. [kafka的listeners和advertised.listeners，配置内外网分流 - 猫尾草 - 掘金](https://juejin.cn/post/6893410969611927566)
2. [Kafka连接服务器出现:Connection to node 1 (localhost/127.0.0.1:9092) could not be established. - 博 荣 - CSDN](https://blog.csdn.net/pbrlovejava/article/details/103451302)

listeners 不能设置为外网 ip，需要设置为 ifconfig 看到的内网 ip。如果 kafka broker 运行在服务器上，请勿设置为本机内网 ip，请继续往下阅读本问题。

1. listeners 指明 kafka 当前节点监听什么来源的 ip
2. advertised.listeners 指明客户端通过哪个 ip 可以访问到当前节点

内网 ip 可使用 ifconfig 查看

```bash
ifconfig
# eth0: 10.217.0.12
```

kafkaPath/conf/server.properties 修改两个地方：

```properties
# listeners=PLAINTEXT://XXXX:9092
listeners=PLAINTEXT://0.0.0.0:9092
# advertised.listeners=PLAINTEXT://your.host.name:9092，your.host.name 改成域名或外网 ip
advertised.listeners=PLAINTEXT://your.host.name:9092
```

强调一下，如果 kafka broker 运行在服务器上，最关键的地方是：

```properties
# 允许外部端口连接
listeners=PLAINTEXT://0.0.0.0:9092
```

这里不能设置成 localhost，必须设置为 0.0.0.0 允许所有 ip 的访问，否则只有 localhost 服务器自己才能访问，服务器外部无法访问。

#### kafka 上一次未正常关闭导致的报错，未在 zookeeper 删除自己的信息，重新启动时发生冲突

参考：[kafka启动失败分析---随笔 - 老树红枫 - CSDN](https://blog.csdn.net/popping_w/article/details/107491003)



## topic

案例版本：v2.5.1

### 创建 topic

```bash
bin/kafka-topics.sh --create --bootstrap-server node1:9092,node2:9092,node3:9092 --topic demoTopic --partitions 3 --replication-factor 2
```

表示：创建一个 名为 topicName 的 Topic。其中指定分区个数为3，副本个数为2。如果 node 数量只有一个，请把分区数量和副本个数都指定为1。

注意：Topic 名称中一定不要同时出现下划线 (’_’) 和小数点 (’.’)。
WARNING: Due to limitations in metric names, topics with a period (’.’) or underscore(’_’) could collide. To avoid issues ot os best to use either, but not both.

当 kafka 集群只有单个 node 时，样例 command

```bash
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --topic demoTopic --partitions 1 --replication-factor 1
```

备注：TopicCommand.createTopic() 方法负责创建 Topic，其核心逻辑是确定新建 Topic 中有多少个分区及每个分区中的副本如何分配，既支持使用 replica-assignment 参数手动分配，也支持使用 partitions 参数和 replication-factor 参数指定分区个数和副本个数进行自动分配。之后该方法会将副本分配结果写入到 ZooKeeper 中。

注意：Kafka 从 2.2 版本开始将 kafka-topic.sh 脚本中的 −−zookeeper 参数标注为 “过时”，推荐使用 −−bootstrap-server 参数。若读者依旧使用的是 2.1 及以下版本，请将下述的 --bootstrap-server 参数及其值手动替换为 --zookeeper zk1:2181,zk2:2181,zk:2181。一定要注意两者参数值所指向的集群地址是不同的。

### 查看 Topic 列表

```bash
bin/kafka-topics.sh --list --bootstrap-server node1:9092,node2:9092,node3:9092
```

查询出来的结果仅有 Topic 的名称信息。

```bash
# 单个 node 可以用这个命令
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### 查看指定 Topic 明细

```bash
bin/kafka-topics.sh --describe --bootstrap-server node1:9092,node2:9092,node3:9092 --topic topicName
```

PartitionCount：partition 个数。
ReplicationFactor：副本个数。
Partition：partition 编号，从 0 开始递增。
Leader：当前 partition 起作用的 breaker.id。
Replicas: 当前副本数据所在的 breaker.id，是一个列表，排在最前面的其作用。
Isr：当前 kakfa 集群中可用的 breaker.id 列表。

单个 node 可以用这个命令

```bash
bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic demoTopic
```

### 参考文档

[Kafka常用命令之kafka-topics.sh - Ernest.Wu - CSDN](https://blog.csdn.net/qq_29116427/article/details/80202392)



## 生产者

### 进入生产者

```bash
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic demoTopic
```



## 消费者

### 进入消费者

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic demoTopic
```



## broker 机制讲解

[KafKa Broker - 我妻礼弥 - 掘金](https://juejin.cn/post/6983966313794240525)



## kafka:broker、client、spring-kafka版本间的关系

kafka broker 和 kafka client, spring-kafka, spring boot, spring cloud 之间有适配关系，即要保证：

1. kafka broker 和 kafka client 适配
2. pring-kafka 和 kafka client 适配(spring-kafka 内包含着 kafka client，根据所需的 kafka client 选择 spring-kafka 的版本)
3. spring-kafka 和 spring boot 适配
4. spring cloud 和 spring-boot 适配。

强行将不适配的版本搭配会导致启动出现问题，启动时可能出现的问题例如：

1. org.apache.kafka.common.errors.TimeoutException: Timeout expired while fetching topic metadata
2. Unexpected exception during bean creation; nested exception is java.lang.TypeNotPresentException: Type org.springframework.kafka.listener.RecordInterceptor not present

解决方法请见如下顺序：

1. [kafka:broker、client、spring-kafka版本间的关系 -  qq_duhai - CSDN](https://blog.csdn.net/qq_16504067/article/details/108778436)

2. [spring-kafka 和 spring boot 之间的关系](https://spring.io/projects/spring-kafka)

3. [spring cloud 与 spring boot 间的适配关系](https://spring.io/projects/spring-cloud)



## 引用/参考

[在Linux上安装Zookeeper - 二木成林 - CSDN](https://blog.csdn.net/cnds123321/article/details/123621681)

[Linux 安装kafka详细步骤 - 不像程序猿的程序员 - CSDN](https://blog.csdn.net/qq_44538738/article/details/114005478)

[kafka使用时的问题 - codeg - 博客园](https://www.cnblogs.com/darklights/p/13414279.html)

[Kafka常用命令之kafka-topics.sh - Ernest.Wu - CSDN](https://blog.csdn.net/qq_29116427/article/details/80202392)

[zookeeper关闭额外随机端口 - SangBigYe - CSDN](https://blog.csdn.net/CutelittleBo/article/details/123275385)

[KAFKA随机产生JMX 端口指定的问题 - Suntoma - CSDN](https://blog.csdn.net/weixin_40209426/article/details/82217987)

[kafka:broker、client、spring-kafka版本间的关系 -  qq_duhai - CSDN](https://blog.csdn.net/qq_16504067/article/details/108778436)

[SpringBoot解决启动kafka：Connection to node -1 could not be established. Broker may not be available - 野花太放肆 - CSDN](https://blog.csdn.net/yu980219/article/details/112301399)

[kafka的listeners和advertised.listeners，配置内外网分流 - 猫尾草 - 掘金](https://juejin.cn/post/6893410969611927566)

[Kafka连接服务器出现:Connection to node 1 (localhost/127.0.0.1:9092) could not be established. - 博 荣 - CSDN](https://blog.csdn.net/pbrlovejava/article/details/103451302)