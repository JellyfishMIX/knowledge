# Distributed Systems Tracing

## 为什么需要调用链监控？

随着系统规模越来越大，服务数目增加，各微服务间的调用关系也越来越错综复杂，通常一个客户端发起的请求在后端系统中会经过多个不同的微服务调用来协同产生结果并返回，在复杂的微服务架构系统中，几乎每一个前端请求都会形成一条复杂的分布式服务调用链路，在每条链路中任何一个依赖服务出现延迟过高或错误的时候都会引起请求最后的失败，这时候，对于每一个请求，调用链的跟踪就变得越来越重要，通过实现对请求调用的跟踪**可以帮助我们快速发现错误以及监控分析每条链路上的性能瓶颈**，现今业界分布式服务跟踪的理论基础主要来自于 Google 的一篇论文《Dapper, a Large-Scale Distributed Systems Tracing Infrastructure》。

## 应用监控缺失会出现哪些问题

- 线上发布服务后，怎么知道这些服务是正常的
- 大量报错，根源在哪里
- 人工配置错误，查找起来要很久很久
- 会不会最后都把问题归根于“网络问题”
- 任何可能出错的地方都会出错，因此微服务需要监控

## Spring Cloud Sleuth

Spring Cloud Sleuth也是依据Dapper论文而诞生的，它的核心概念是：

- Trace：一次分布式调用的链路踪迹
- Span：一个方法（局部或远程）调用踪迹
- Annotation：附着在Spane上的日志信息
- Sampling：采样率（0-1，1是全采样）
- 引用Sleuth官方提供的示意图（[Sletuh官网地址](https://cloud.spring.io/spring-cloud-sleuth/2.0.x/multi/multi__introduction.html)），将Span和Trace在一个系统中使用Zipkin注解的过程图形化：

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201012181323.png)


![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201012181330.png)

The following image shows how parent-child relationships of spans look:

![Parent child relationship](https://image-hosting.jellyfishmix.com/20201012181343.png)

## Zipkin

Zipkin 是一个开放源代码分布式的跟踪系统，由Twitter公司开源，它致力于收集服务的定时数据，以解决微服务架构中的延迟问题，包括数据的收集、存储、查找和展现，Zipkin的设计是基于谷歌的Google Dapper论文。
Zipkin的服务端已经打包成了一个 jar，使用 java -jar zipkin-server.jar 启动，访问 localhost:9411 查看zipkin主页

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201012181538.png)

Zipkin官网地址：https://zipkin.io/

## Spring Cloud Sleuth 与 Zipkin 的关系

Zipkin收集 Sleuth 产生的数据，并以界面的形式呈现出来。

## Configuration

Dependency

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```

## Example

spring-cloud-sleuth

output of console

```
2020-10-12 16:34:39.294  INFO [brother-takeaway-order,81492de68d74b809,716dccc0c8414fd2,false] 25626 --- [ntContainer#3-1] m.j.b.order.message.ProductInfoReceiver  : 接收到消息...
```

description:

[brother-takeaway-order,81492de68d74b809,716dccc0c8414fd2,false]

brother-takeaway-order: service name(spring.application.name)

81492de68d74b809: traceId

716dccc0c8414fd2: spanId

false/true: whether to output this information to other services to display.

## Else Distributed System Tracing

在团队小，业务量小时，为了快速开发，使用 Spring Cloud Sleuth + Zipkin可以快速进行微服务监控，但是 Zipkin 有些缺点

美团-大众点评在git上开源了 CAT 监控工具，有使用教程，超过8000课星，有业界一线的互联网公司接入（很成熟了，一些坑已经被踩了），在服务规模大时，接入 CAT 监控是个不错的选择。
CAT 地址：https://github.com/dianping/cat
![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201012184041.png)
网友对 CAT 、Zipkin、Pinpoint等监控组件进行了对比
![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201012184048.png)

## 引用/参考

[SpringCloud实战十六：Spring Cloud Sleuth + Zipkin 调用链监控 - 闪耀的瞬间 - CSDN](https://blog.csdn.net/zhuyu19911016520/article/details/87181804)