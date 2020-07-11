# Spring Cloud Stream



## 前言

[Spring Cloud Stream](https://spring.io/projects/spring-cloud-stream) 在 Spring Cloud 体系内用于构建高度可扩展的基于事件驱动的微服务。

Spring Cloud Stream 本身内容很多，而且它还有很多外部的依赖，想要熟悉 Spring Cloud Stream，需要了解以下知识：

- Spring Framework(Spring Messaging, Spring Environment)
- Spring Boot Actuator
- Spring Boot Externalized Configuration
- Spring Retry
- Spring Integration
- Spring Cloud Stream

这篇文章的目的主要是介绍 Spring Cloud Stream。面对这么多知识点，会尽量以最简单的方式带大家了解 Spring Cloud Steam(后面会以 SCS 代替 Spring Cloud Stream)。

了解 SCS 之前，必须要了解 Spring Messaging 和 Spring Integration 这两个项目，首先我们看下这两个项目。



## Spring Messaging

Spring Messaging 模块是 Spring Framework 中的一个模块，其作用就是统一消息的编程模型。

比如 `Message`对应的模型就包括一个消息头 `Header` 和消息体`Payload`

![img](https://image-hosting.jellyfishmix.com/20200627222610.jpg)

```java
package org.springframework.messaging;
public interface Message<T> {
	T getPayload();
	MessageHeaders getHeaders();
}
```

消息通道`MessageChannel`用于接收消息，调用 `send` 方法可以将消息发送至该消息通道中：

![img](https://image-hosting.jellyfishmix.com/20200627222725.jpg)

```java
@FunctionalInterface
public interface MessageChannel {
	long INDEFINITE_TIMEOUT = -1;
	default boolean send(Message<?> message) {
		return send(message, INDEFINITE_TIMEOUT);
	}
	boolean send(Message<?> message, long timeout);
}
```

`MessageChannel`里的 `Message`如何被消费呢？ 由`MessageChannel`的子接口`SubscribableChannel` 实现，被 `MessageHandler` 所订阅：

```java
public interface SubscribableChannel extends MessageChannel {
	boolean subscribe(MessageHandler handler);
	boolean unsubscribe(MessageHandler handler);
}
```

`MessageHandler` 真正地消费/处理消息：

```java
@FunctionalInterface
public interface MessageHandler {
	void handleMessage(Message<?> message) throws MessagingException;
}
```

Spring Messaging 内部在消息模型的基础上衍生出了其它的一些功能，如：

1. 消息接收参数及返回值处理：消息接收参数处理器 `HandlerMethodArgumentResolver` 配合 `@Header`, `@Payload` 等注解使用；消息接收后的返回值处理器 `HandlerMethodReturnValueHandler` 配合 `@SendTo` 注解使用
2. 消息体内容转换器 `MessageConverter`
3. 统一抽象的消息发送模板 `AbstractMessageSendingTemplate`
4. 消息通道拦截器 `ChannelInterceptor`
5. …



## Spring Integration

[Spring Integration](https://docs.spring.io/spring-integration/reference/html/index.html) 提供了 Spring 编程模型的扩展用来支持企业集成模式([Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/))。

Spring Integration 是对 Spring Messaging 的扩展。它提出了不少新的概念，包括消息的路由 `MessageRoute`、消息的分发 `MessageDispatcher`、消息的过滤 `Filter`、消息的转换 `Transformer`、消息的聚合 `Aggregator`、消息的分割 `Splitter` 等等。同时还提供了包括 `MessageChannel` 的实现 `DirectChannel`、`ExecutorChannel`、`PublishSubscribeChannel` 等; `MessageHandler` 的实现 `MessageFilter`、`ServiceActivatingHandler`、`MethodInvokingSplitter` 等内容。



### 几种消息的处理方式

#### 消息的分割

![img](https://image-hosting.jellyfishmix.com/20200627225451.png)

#### 消息的聚合

![img](https://image-hosting.jellyfishmix.com/20200627225522.png)

#### 消息的过滤

![img](https://image-hosting.jellyfishmix.com/20200627225550.png)

#### 消息的分发

![img](https://image-hosting.jellyfishmix.com/20200627225647.png)

我们以一个最简单的例子来尝试一下 Spring Integration

```java
SubscribableChannel messageChannel = new DirectChannel(); // 1

messageChannel.subscribe(msg -> { // 2
  System.out.println("receive: " + msg.getPayload());
});

messageChannel.send(MessageBuilder.withPayload("msg from alibaba").build()); // 3
```

这段代码解释：

1. 构造一个可订阅的消息通道 `messageChannel`
2. 使用 `MessageHandler` 去消费这个消息通道里的消息
3. 发送一条消息到这个消息通道，消息最终被消息通道里的 `MessageHandler` 所消费

最后控制台打印出: `receive: msg from alibaba`

`DirectChannel` 内部有个 `UnicastingDispatcher` 类型的消息分发器，会分发到对应的消息通道 `MessageChannel` 中，从名字也可以看出来，`UnicastingDispatcher` 是个单播的分发器，只能选择一个消息通道。那么如何选择呢? 内部提供了 `LoadBalancingStrategy`负载均衡策略，默认只有轮询的实现，可以进行扩展。



### Spring Cloud Stream

SCS 在 Spring Integration 的基础上进行了封装，提出了 `Binder`, `Binding`, `@EnableBinding`, `@StreamListener` 等概念; 与 Spring Boot Actuator 整合，提供了 `/bindings`, `/channels` endpoint; 与 Spring Boot Externalized Configuration 整合，提供了 `BindingProperties`, `BinderProperties` 等外部化配置类; 增强了消息发送失败的和消费失败情况下的处理逻辑等功能。

SCS 是 Spring Integration 的加强，同时与 Spring Boot 体系进行了融合，也是 Spring Cloud Bus 的基础。它屏蔽了底层消息中间件的实现细节，希望以统一的一套 API 来进行消息的发送/消费，底层消息中间件的实现细节由各消息中间件的 Binder 完成。

![简介](https://image-hosting.jellyfishmix.com/20200627230843.png)

![img](https://image-hosting.jellyfishmix.com/20200627231246.png)

`Binder` 是提供与外部消息中间件集成的组件，会构造 `Binding`，提供了 2 个方法分别是 `bindConsumer` 和 `bindProducer` 分别用于构造生产者和消费者。目前Spring Cloud官方的实现有 [Rabbit Binder](https://github.com/spring-cloud/spring-cloud-stream-binder-rabbit) 和 [Kafka Binder](https://github.com/spring-cloud/spring-cloud-stream-binder-kafka)， [Spring Cloud Alibaba](https://github.com/spring-cloud-incubator/spring-cloud-alibaba) 内部实现了 [RocketMQ Binder](https://github.com/spring-cloud-incubator/spring-cloud-alibaba/tree/master/spring-cloud-stream-binder-rocketmq)。

`Binding` 从图中可以看出，它是连接应用程序跟消息中间件的桥梁，用于消息的消费和生产。



## 结合SpringBoot使用SpringCloudStream

```yaml
spring:
  stream:
    bindings:
      myMessage:
      	# 消息分组，避免多个服务实例同时接收到同样的消息
        group: order
        # 与RabbitMQ传输信息的时候，自动序列化/反序列化为json对象
        content-type: application/json
```

StreamClient.java

```java
public interface StreamClient {
    String EXCHANGE = "myMessage";
    String EXCHANGE_FEEDBACK = "feedback";

    // myMessage是绑定的Exchange名称。启动后自动创建一个Queue。
    @Input(StreamClient.EXCHANGE)
    SubscribableChannel input();

    // myMessage是绑定的Exchange名称。启动后自动创建一个Queue。
    @Output(StreamClient.EXCHANGE)
    MessageChannel output();

    // feedback是绑定的Exchange名称。启动后自动创建一个Queue。
    @Input(StreamClient.EXCHANGE_FEEDBACK)
    SubscribableChannel feedbackInput();

    // feedback是绑定的Exchange名称。启动后自动创建一个Queue。
    @Output(StreamClient.EXCHANGE_FEEDBACK)
    MessageChannel feedbackOutput();
}
```

StreamReceiver.java

```java
/**
 * Spring Cloud Stream测试
 */
@Component
@EnableBinding(StreamClient.class)
@Slf4j
public class StreamReceiver {
    // @SteamListener默认状态下，不能有多个方法绑定同一个Channel，这样Channel会无法判断把Message送往哪个方法（初学时写下此注释，此说法存疑）

    // /**
    //  * 监听字符串信息
    //  *
    //  * @param message message
    //  */
    // @StreamListener(value = StreamClient.EXCHANGE)
    // public void processString(Object message) {
    //     log.info("StreamReceiver: {}", message);
    // }

    /**
     * 监听OrderDTO信息，并使用@SendTo给出反馈
     *
     * @param message message
     * @return
     */
    @StreamListener(value = StreamClient.EXCHANGE)
    @SendTo(StreamClient.EXCHANGE_FEEDBACK)
    public String processObject(OrderDTO message) {
        log.info("StreamReceiver: {}", message);
        return "received";
    }

    /**
     * 监听反馈
     *
     * @param message message
     */
    @StreamListener(value = StreamClient.EXCHANGE_FEEDBACK)
    public void processResponse(String message) {
        log.info("Feedback: " + message);
    }
}
```

StreamSenderTest.java

```java
/**
 * 发送Mq消息测试
 */
public class StreamSenderTest extends BrotherTakeawayOrderApplicationTests {
    @Resource
    private StreamClient streamClient;

    @Test
    @Disabled
    void sendMessage() {
        String message = "now: " + new Date();
        streamClient.output().send(MessageBuilder.withPayload(message).build());
    }

    @Test
    @Disabled
    void sendObject() {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setOrderId("123456");
        streamClient.output().send(MessageBuilder.withPayload(orderDTO).build());
    }
}
```



## 引用/参考

[Spring Cloud Stream 体系及原理介绍 - Format's Notes](https://fangjian0423.github.io/2019/04/03/spring-cloud-stream-intro/)