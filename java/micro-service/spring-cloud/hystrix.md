# hystrix



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
@DefaultProperties
public class HystrixController {
    @HystrixCommand(fallbackMethod = "fallback")
    @GetMapping("/product-list")
    public String getProductInfoList() {
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.getForObject("http://localhost:8082/product/list", String.class);
    }

    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
    })
    @GetMapping("/product-list2")
    public String getProductInfoList2() {
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

- docker通过仓闭模式实现进程隔离，使得容器之间互不影响

- 而Hystrix使用仓闭模式实现的是线程池隔离，会为每个Hystrix command创建独立的线程池，这样就算某个Hystrix command包装下的服务出现延迟过高的情况，也只会对该依赖服务的调用产生影响，并不会影响其它服务。

