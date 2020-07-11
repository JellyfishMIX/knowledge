# RabbitMQ with SpringBoot



## 配置

application.yml

```yaml
spring:
    rabbitmq:
        host: localhost
        port: 5672
        username: guest
        password: guest
```

pom.xml

```xml
<!--使用RabbitMQ进行通信-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```



## 注解

在接收方使用 `@RabbitListener` 注解后，会自动创建Exchange和Queue。



## 使用

MqReceiver.java

```java
@Slf4j
@Component
public class MqReceiver {
    // 1. 手动创建mq
    // @RabbitListener(queues = "myQueue")
    // 2. 自动创建mq
    // @RabbitListener(queuesToDeclare = @Queue("myQueue"))
    // 3. 自动创建队列，Exchange和Queue绑定
    @RabbitListener(bindings = @QueueBinding(
            exchange = @Exchange("myExchange"),
            value = @Queue("myQueue")
    ))
    public void process(String message) {
        log.info("computer MqReceiver: {}", message);
    }

    @RabbitListener(bindings = @QueueBinding(
            exchange = @Exchange("myOrder"),
            // RoutingKey
            key = "computer",
            value = @Queue("computerOrder")
    ))
    public void processComputer(String message) {
        log.info("computer MqReceiver: {}", message);
    }

    @RabbitListener(bindings = @QueueBinding(
            exchange = @Exchange("myOrder"),
            // RoutingKey
            key = "fruit",
            value = @Queue("fruitOrder")
    ))
    public void processFruit(String message) {
        log.info("fruit MqReceiver: {}", message);
    }
}
```

MqSenderTest.java

```java
public class MqSenderTest extends BrotherTakeawayOrderApplicationTests {
    @Autowired
    private AmqpTemplate amqpTemplate;

    @Test
    @Disabled
    void send() {
        System.out.println("--- test begin");
        // 第一个参数：exchange，第二个参数message
        amqpTemplate.convertAndSend("myQueue", "now: " + new Date());
    }

    @Test
    @Disabled
    void sendOrder() {
        // 第一个参数：exchange，第二个参数：routingKey，第三个参数：message
        amqpTemplate.convertAndSend("myOrder", "computer", "now: " + new Date());
    }
}
```

