## todo-learn



## 业务代码

1. 分析 ota 计算引擎中 CaffeineCache 的使用。



## jdk

1. Optional 源码分析 ok
2. jdk 8 CompletableFuture
3. ConcurrentHashMap
4. 各种集合
5. enum 枚举的原理。



## spring-ioc

1. BeanDefinition

2. BeanDefinitionBuilder

3. BeanFactory

4. AbstractBeanFactory

5. FactoryBean

6. 注册 bean 的方式

7. Spring容器中获取Bean实例的七种方式

8. Aware 接口。ApplicationContextAware。

9. List 如何注入，List 中的泛型如何得到真实类型？org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveDependency -> org.springframework.beans.factory.support.DefaultListableBeanFactory#doResolveDependency -> org.springframework.beans.factory.support.DefaultListableBeanFactory#resolveMultipleBeans -> org.springframework.beans.factory.config.DependencyDescriptor#getDependencyType

10. BeanPostProcessor 体系



## spring-mvc

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle

org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite#handleReturnValue

org.springframework.web.method.support.HandlerMethodReturnValueHandler#handleReturnValue

org.springframework.web.context.request.async.WebAsyncUtils#getAsyncManager(org.springframework.web.context.request.WebRequest)

org.springframework.web.context.request.async.WebAsyncManager#startDeferredResultProcessing

org.springframework.web.context.request.async.WebAsyncManager#startAsyncProcessing

org.springframework.web.context.request.async.AsyncWebRequest#startAsync

org.springframework.util.Assert#state(boolean, java.lang.String)

javax.servlet.ServletRequest#isAsyncSupported

CoyoteRequest.asyncSupported: false, 会抛出异常: 

```
java.lang.IllegalStateException: Async support must be enabled on a servlet and for all filters involved in async request processing. This is done in Java code using the Servlet API or by adding "<async-supported>true</async-supported>" to servlet and filter declarations in web.xml.
```



TransactionDefinition



ExceptionHandler 的运行机制？ExceptionHandlerMethodResolver 源码解析。

1. SpringMVC 如何序列化反序列化的？
   1. Spring Boot 中默认使用的 Json 解析技术框架是jackson，可以换成其他组件，比如FastJson。
   2. 如何替换？背后可插拔的机制是什么样的。

spring aop aspect 如何生效的，背后的机制？

bean 的 alias 别名有什么用？

@Nullable 标签



InstantiationAwareBeanPostProcessor.java

BeanPostProcessor

三级缓存。



spring 有几种创建对象的方式？



## spring-aop

1. 切面，切点，切片，织入等系统化概念要能讲出来，不是说实际 aop 怎么做。



## 响应式编程

使用样例: com.qunar.flight.international.worker.ota.paoding.controller.PaoDingController#debugSearch

Mono.delay 超时机制。



## maven

1. 为什么 spring boot 项目的 pom.xml 中 dependency 不用写版本就能生效？
2. maven pom.xml 的 version 1.0-SNAPSHOT 中 SNAPSHOT 有什么作用？
3. maven dependency 的 scope，如果是 complie 或 test，在正式运行字节码的时候，没有引入 jar 包，会有缺少注解或类错误吗？
4. junit 如何运行的？



## dubbo

dubbo 依赖了 netty，了解到 dubbo 底层使用 netty 进行通信。但是 dubbo 是作为依赖导入的，也需要 spring-web 包，spring web 包中有 springMVC，springMVC 不是在 tomcat 后面流程作为 servlet 去运行的吗，而且 spirng boot 还内置了 tomcat。所以 dubbo 把 tomcat 移除了吗，如何移除的？

dubbo provider 的代码没有运行在 tomcat 里，dubbo 内置了一个 netty，provider 的代码在 netty 中运行。

dubbo 中的 aop 机制。



## guava

1. guava AbstractFuture
2. guava MoreExecutors
3. list, set 切分
4. ImmutableList
5. Splitter, Joiner



## apache common3

1. Pair
2. MutablePair
3. ImmutablePair



## juc 包

Condition

CountDownLuntch



## kafka

1. kafka 既然是 consumer 主动拉取，为什么还会触发拒绝策略? 影响 consumer 拉取速度的因素? 为什么都拉取到触发拒绝策略了，consumer 还在拉取?



## 日志

1. `protected final Log logger = LogFactory.getLog(getClass());` 经常出现的这种获取 loggger 的代码，到底做了什么？



## netty

netty 作为一个高性能的通信框架，值得看源码。



## JVM 调优/JVM 启动参数

面试可能会问，看一下。



## outlook 附带的 microsoft auto update 移除它

这个东西干扰 doing，移除它。



## zookeeper

zookeeper 源码用 java 写的，zookeeper 客户端如何注册自身 ip 的？



## 其他

1. github 的 pagehelper 如何做到分页的？为什么只有紧跟着 PageHelper.startPage() 的那一句起了作用？
2. druid 源码分析
3. jar 命令如何找到 main 方法。
4. 分布式中的强一致性如何保证？增量同步，全量同步。
5. qunar 安全课程，常见的攻击方式。