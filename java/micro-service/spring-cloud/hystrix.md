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