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