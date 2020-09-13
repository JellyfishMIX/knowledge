# hystrix



## 雪崩效应

微服务中某个服务不可用导致级联故障，最终造成整个系统不可用的情况。

![image-20201003165925915](https://image-hosting.jellyfishmix.com/20201003165925.png)

e.g.

A调用B，B调用C。

C故障，B调用C请求不成功多次重试，造成资源耗尽，导致B不可用。

B故障，A调用B请求不成功多次重试，造成资源耗尽，导致A不可用。



## 基本信息

防止雪崩利器，熔断器。



## 功能

- 服务降级
  - 优先核心业务，非核心业务不可用或弱可用
  - 通过HystrixCommand注解指定
  - fallbackMethod（回退函数）中具体实现降级逻辑

- 依赖隔离
- 服务熔断
- 监控（Hystrix Dashboard）



## 使用

### 服务降级

```java
@RequestMapping("/hystrix")
@RestController
@DefaultProperties(defaultFallback = "/defaultFallback")
public class HystrixController {
    @HystrixCommand(fallbackMethod = "fallback")
    @GetMapping("/product-list")
    public String getProductInfoList() {
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject("http://localhost:8082/product/list", String.class);
    }

    /**
     * Only add @HystrixCommand annotation.
     * When the service degradation is triggered,
     * the defaultFallback method specified in the @DefaultProperties annotation of the class of the method is called.
     * 只添加@HystrixCommand注解，触发服务降级时会调用所处类的@DefaultProperties注解中指定的defaultFallback方法
     * @return
     */
    @HystrixCommand
    @GetMapping("/product-list2")
    public String getProductInfoList2() {
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject("http://localhost:8082/product/list", String.class);
    }

    @HystrixCommand(commandProperties = {
            // 超时时间
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
    })
    @GetMapping("/product-list3")
    public String getProductInfoList3() {
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject("http://localhost:8082/product/list", String.class);
    }

    private String fallback() {
        return "太拥挤了，请稍后再试";
    }

    private String defaultFallback() {
        return "默认提示：太拥挤了，请稍后再试";
    }
}
```

- 当服务不可用时，会触发 `fallbackMethod` 所指定的方法。

- 添加@DefaultProperties注解后，服务不可用会触发 `defaultFallback()` 方法。

### 依赖隔离

- docker通过舱壁模式实现进程隔离，使得容器之间互不影响。
- 而Hystrix使用舱壁模式实现的是线程池隔离，会为每个Hystrix command创建独立的线程池，这样就算某个Hystrix command包装下的服务出现延迟过高的情况，也只会对该依赖服务的调用产生影响，并不会影响其它服务。



## 服务熔断

熔断器状态机

![state](https://image-hosting.jellyfishmix.com/20201003234909.png)



滚动时间窗口：断路器在关闭状态时，统计一些请求和错误数据，以确定是否需要打开时，有时间范围（即隔一段时间统计一次），此时间范围被称为"时间窗口"。

当断路器打开，对主逻辑进行熔断后，会启动一个休眠时间窗，在此时间窗内，降级逻辑临时成为主逻辑。休眠时间窗到期，断路器进入半开状态，释放一次请求到原来的主逻辑上，如果此次请求正常返回，断路器闭合，主逻辑恢复。如果此次请求依然有问题，断路器将进入打开状态，休眠时间窗重新计时。

e.g.

```java
@RequestMapping("/hystrix")
@RestController
@DefaultProperties(defaultFallback = "/defaultFallback")
public class HystrixController {
    @HystrixCommand(commandProperties = {
            // 开启熔断
            @HystrixProperty(name = "circuitBreaker.enable", value = "true"),
            // 滚动时间窗口中，断路器的进行统计的最小请求数
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            // 休眠时间窗到期时间
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"),
            // 滚动时间窗口中，断路器打开的错误百分比条件
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60")
    })
    @GetMapping("/product-list4")
    public String getProductInfoList4() {
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject("http://localhost:8082/product/list", String.class);
    }

    private String defaultFallback() {
        return "默认提示：太拥挤了，请稍后再试";
    }
}
```

关于熔断器（circuit breaker），详细可以阅读Martin Fowler的文章：[CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)



## Hystrix Dashboard

- import two dependency

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
      <version>1.4.7.RELEASE</version>
  </dependency>
  <!-- 如果已经有了，不需要再引入。例如spring-cloud-starter-stream-rabbit中含有此dependency -->
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
      <version>2.3.1.RELEASE</version>
  </dependency>
  ```

- Annotate the startup class with @EnableHystrixDashboar