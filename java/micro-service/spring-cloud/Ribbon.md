# Ribbon



## 简介

Ribbon是Spring Cloud的服务端发现负载均衡组件，改进自Netflix Ribbon。

核心功能有三点：

- 服务端发现

  发现服务端的服务列表

- 服务监听

  检测失效的服务，做到高效剔除

- 服务选择规则

  选择使用的负载均衡规则

主要的组件及流程：

1. ServerList

   获取所有可用服务列表

2. ServerListFilter

   过滤掉无效的地址

3. IRule

   通过策略选择一个实例，作为最终目标结果



## IRule修改策略

application.yml

```
users:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```

Reference:

https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.1.RELEASE/reference/html/#customizing-the-ribbon-client-by-setting-properties

