# Feign



## 简介

Feign 是一个声明式的web服务客户端，这便得编写web服务客户端更容易，使用Feign 创建一个接口并对它进行注解，它具有可插拔的注解支持包括Feign注解与JAX-RS注解，Feign还支持可插拔的编码器与解码器，Spring Cloud 增加了对 Spring MVC的注解，Spring Web 默认使用了HttpMessageConverters, Spring Cloud 集成 Ribbon 和 Eureka 提供的负载均衡的HTTP客户端 Feign.

- 声明式REST客户端（伪RPC）

- 采用了接口 + 注解的方式
- Feign内部使用了Ribbon做负载均衡

