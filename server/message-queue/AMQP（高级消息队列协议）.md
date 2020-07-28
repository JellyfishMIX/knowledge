# AMQP（高级消息队列协议）



## 概念

![img](https://image-hosting.jellyfishmix.com/20200627100105.jpg)

从 AMQP 协议可以看出，Queue、Exchange 和 Binding 构成了 AMQP 协议的核心

- Producer：消息生产者，即投递消息的程序。

- Broker：消息队列服务器实体。

- - Exchange：消息交换机，它指定消息按什么规则，路由到哪个队列。
  - Binding：绑定，它的作用就是把 Exchange 和 Queue 按照路由规则绑定起来。
  - Queue：消息队列载体，每个消息都会被投入到一个或多个队列。

- Consumer：消息消费者，即接受消息的程序。



## Exchange

### 介绍

为什么我们需要 Exchange 而不是直接将Message发送至Queue呢？

AMQP 协议中的核心思想是**Producer和Consumer解耦**，Producer从不直接将Message发送给Queue。Producer通常不知道是否一个Message会被发送到Queue中，只是将Message发送到一个Exchange。

先由 Exchange 来接收，然后 Exchange 按照特定的策略转发到 Queue 进行存储。Exchange 就类似于一个交换机，将各个消息分发到相应的队列中。

在实际应用中我们只需要定义好 Exchange 的路由策略，而Produer则不需要关心Message会发送到哪个 Queue 或被哪些 Consumer 消费。在这种模式下Producer只面向 Exchange 发布消息，Consumer只面向 Queue 消费Message，Exchange 定义了消息路由到 Queue 的规则，将各个层面的消息传递隔离开，使每一层只需要关心自己面向的下一层，降低了整体的耦合度。



### 理解Exchange

Exchange 收到消息时，他是如何知道需要发送至哪些 Queue 呢？这里就需要了解 Binding 和 RoutingKey 的概念：

Binding 表示 Exchange 与 Queue 之间的关系，我们也可以简单的认为队列对该交换机上的消息感兴趣，绑定可以附带一个额外的参数 RoutingKey。Exchange 就是根据这个 RoutingKey 和当前 Exchange 所有绑定的 Binding 做匹配，如果满足匹配，就往 Exchange 所绑定的 Queue 发送消息，这样就解决了我们向 RabbitMQ 发送一次消息，可以分发到不同的 Queue。RoutingKey 的意义依赖于交换机的类型。

下面就来了解一下 Exchange 的三种主要类型：`Fanout`、`Direct` 和 `Topic`。



### Fanout Exchange

#### 介绍

![img](https://image-hosting.jellyfishmix.com/20200627141857.jpg)

Fanout Exchange 会忽略 RoutingKey 的设置，直接将 Message 广播到所有绑定的 Queue 中。

#### 应用场景

以日志系统为例：假设我们定义了一个 Exchange 来接收日志消息，同时定义了两个 Queue 来存储消息：一个记录将被打印到控制台的日志消息；另一个记录将被写入磁盘文件的日志消息。我们希望 Exchange 接收到的每一条消息都会同时被转发到两个 Queue，这种场景下就可以使用 Fanout Exchange 来广播消息到所有绑定的 Queue。



### Direct Exchange

#### 介绍

![img](https://image-hosting.jellyfishmix.com/20200627142058.jpg)

Direct Exchange 是 RabbitMQ 默认的 Exchange，完全根据 RoutingKey 来路由消息。设置 Exchange 和 Queue 的 Binding 时需指定 RoutingKey（一般为 Queue Name），发消息时也指定一样的 RoutingKey，消息就会被路由到对应的Queue。

#### 应用场景

现在我们考虑只把重要的日志消息写入磁盘文件，例如只把 Error 级别的日志发送给负责记录写入磁盘文件的 Queue。这种场景下我们可以使用指定的 RoutingKey（例如 error）将写入磁盘文件的 Queue 绑定到 Direct Exchange 上。



### Topic Exchange

#### 介绍

![img](https://image-hosting.jellyfishmix.com/20200627142149.jpg)



Topic Exchange 和 Direct Exchange 类似，也需要通过 RoutingKey 来路由消息，区别在于Direct Exchange 对 RoutingKey 是精确匹配，而 Topic Exchange 支持模糊匹配。分别支持 `*` 和 `#` 通配符，`*` 表示匹配一个单词，`#` 则表示匹配没有或者多个单词。

#### 应用场景

假设我们的消息路由规则除了需要根据日志级别来分发之外还需要根据消息来源分发，可以将 RoutingKey 定义为 `消息来源.级别` 如 `order.info`、`user.error` 等。处理所有来源为 `user ` 的 Queue 就可以通过 `user.*` 绑定到 Topic Exchange 上，而处理所有日志级别为 `info` 的 Queue 可以通过 `*.info` 绑定到 Exchange上。



### 两种特殊的 Exchange

#### Headers Exchange

Headers Exchange 会忽略 RoutingKey 而根据消息中的 Headers 和创建绑定关系时指定的 Arguments 来匹配决定路由到哪些 Queue。

Headers Exchange 的性能比较差，而且 Direct Exchange 完全可以代替它，所以不建议使用。

#### Default Exchange

Default Exchange 是一种特殊的 Direct Exchange。当你手动创建一个队列时，后台会自动将这个队列绑定到一个名称为空的 Direct Exchange 上，绑定 RoutingKey 与队列名称相同。有了这个默认的交换机和绑定，使我们只关心队列这一层即可，这个比较适合做一些简单的应用。



### Exchange总结

在 Exchange 的基础上我们可以通过比较简单的配置绑定关系来灵活的使用消息路由，在简单的应用中也可以直接使用 RabbitMQ 提供的 Default Exchange 而无需关心 Exchange 和绑定关系。Direct Exchange、Topic Exchange、Fanout Exchange 三种类型的交换机的使用方式也很简单，容易掌握。



## 引用/参考

 [理解 RabbitMQ Exchange - 小芳芳的文章 - 知乎](https://zhuanlan.zhihu.com/p/37198933)

