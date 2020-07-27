# zuul



## 基本信息

基于Spring Cloud生态的RESTFul API网关

### 特点

- 路由+过滤器
- 核心是一些列的过滤器



## 架构图

![image-20200718161034215](https://image-hosting.jellyfishmix.com/20200718161034.png)

- 过滤器之间没有直接通信。
- 过滤器之间通过Request Context进行数据传递。



## 过滤器API

- 前置(Pre)
- 路由(Route)
- 后置(Post)
- 错误(Error)



## 请求生命周期

![image-20200718161755800](https://image-hosting.jellyfishmix.com/20200718161755.png)



## 自定义路由

```yaml
zuul:
  # 自定义路由，所有路由查看可访问endpoint: /actuator/routes
  routes:
    # route映射，把 /brother-takeaway-product/** 映射到 /myProduct/**，并且/brother-takeaway-product/**依然保留
     myProduct:
       path: /myProduct/**
       serviceId: brother-takeaway-product
       # sensitiveHeaders，设置需要被过滤的request header敏感信息，cookies默认被zuul过滤。置空表示不设置需要被过滤的request header敏感信息。
       sensitiveHeaders:
    # 简洁写法（简洁写法无法设置sensitiveHeaders）
    # brother-takeaway-product: /myProduct/**
  # 正则模式设置禁止访问的路由
  ignored-patterns:
    - /brother-takeaway-product/product/list
    - /myProduct/product/list
    # 通配符写法，会禁止所有的/**/product/list路由
    # - /**/product/list

# 设置路由端点，查看json
# actuator 启用所有的监控端点 “*”号代表启用所有的监控端点，可以单独启用，例如，health，info，metrics
# spring boot 升为 2.0 后，为了安全，默认 Actuator 只暴露了2个端点，heath 和 info
management:
  endpoints:
    web:
      exposure:
        include: routes
```



## zuul配置动态刷新

```java
@Component
public class ZuulConfig {
    @Bean
    @Primary
    @ConfigurationProperties("zuul")
    @RefreshScope
    public ZuulProperties zuulProperties() {
        return new ZuulProperties();
    }
}
```



## 典型应用场景

### 前置（Pre）

- 限流

- 鉴权

- 参数校验调整

- 请求转发

### 后置（Post）

- 统计

- 日志
- 跨域



## zuul的高可用

- 多个zuul节点注册到Eureka Server

- nginx + zuul混搭

nginx转发到多个zuul服务，nginx继续做负载均衡，和zuul互相取长补短。



## zuul的跨域

- 在被调用的类或方法上增加@CrossOrigin注解
- 在Zuul里增加CorsFilter过滤器

