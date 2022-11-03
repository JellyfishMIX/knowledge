# dubbo 服务发布相关源码分析



## dubbo 与 spring 的整合--dubbo 的 xml 配置

在 spring 中使用 dubbo 的方式有两种，一种是通过 xml 配置文件，一种是通过注解的方式。

在 Spring 中提供了一种 NamespaceHandler 的机制，用于对 Spring 的标签进行扩展。

通过 spring xml 的方式使用 dubbo 时，dubbo 会提供一个名为 DubboNamespaceHandler 的处理器，用于解析 spring 的 xml 配置中的各种 dubbo 标签，并注入到容器中，这里不再深入。



## dubbo-provider 的配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
	http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="demo-provider"/>
    <!-- 使用 multicast 广播注册中心暴露服务地址 -->
    <dubbo:registry address="zookeeper://localhost:2181"/>
    <!-- 使用 dubbo 协议在 20880 端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!-- 声明需要暴露的服务接口 -->
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>
    <!-- 和本地 bean 一样实现服务 -->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
</beans>
```



## DubboNamespaceHandler

org.apache.dubbo.config.spring.schema.DubboNamespaceHandler

```java
/**
 * DubboNamespaceHandler
 *
 * @export
 */
public class DubboNamespaceHandler extends NamespaceHandlerSupport implements ConfigurableSourceBeanMetadataElement {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class, true));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class, true));
        registerBeanDefinitionParser("ssl", new DubboBeanDefinitionParser(SslConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, true));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

这里面初始化了 dubbo 配置文件，可以看到有各种 dubbo xml 配置中可能用到的标签。服务发布相关逻辑在 ServiceBean 中。



## Invoker

Invoker，一个可执行对象，能根据方法名称，参数得到相应的执行结果。接口如下：

```java
public interface Invoker<T> {
    Class<T> getInterface();

    URL getUrl();

    Result invoke(Invocation invocation) throws RpcException;

    void destroy();
}
```

Invocation  接口包含了需要执行的方法，参数等信息，接口如下：

```java
public interface Invocation {
    URL getUrl();

    String getMethodName();

    Class<?>[] getParameterTypes();

    Object[] getArguments();
}
```

目前实现类只有一个 RpcInvocation

Invoker 执行过程分成三种类型：

1. 本地执行的 Local Invoker。
2. 远程通信执行的 Remote Invoker。
3. 多个 Remote Invoker 聚合成的集群版 Invoker。

以HelloService接口为例：

本地执行的 Local Invoker: server 端，含有对应的 HelloServiceImpl 实现，要执行该接口方法，只需要通过反射执行HelloServiceImpl对应的方法即可。

远程通信执行的 Remote Invoker: client 端执行该接口的方法，需要进行远程通信，发送要执行的参数信息给 server 端。server 端利用本地执行的 Local Invoker 执行相应的方法，然后将执行结果返回给 client 端。这个过程是经典的 rpc 调用。

集群 Invoker：client 端拥有某个服务的多个 Invoker，将多个 Invoker 聚合成一个集群版的 Invoker，client 端使用时，只需通过集群版的Invoker来进行操作。集群版的Invoker会从众多的远程通信类型的Invoker中选择一个来执行（从中加入负载均衡、服务降级等策略），类似服务治理，dubbo已经实现了



## 服务发布机制介绍

![这里写图片描述](https://image-hosting.jellyfishmix.com/20221101013949.jpeg)

1. 首先 ServiceConfig 拿到对外提供服务的实现类 ref(如: HelloWorldImpl)，然后通过 ProxyFactory 类的 getInvoker 方法使用 ref 生成一个 AbstractProxyInvoker 实例，完成具体服务到 Invoker 的转化。接下来就是 Invoker 转换到 Exporter 的过程。

2. Dubbo处理服务暴露的关键就在Invoker转换到Exporter的过程(如上图中的红色部分)，Dubbo协议的Invoker转为Exporter发生在DubboProtocol类的export方法，它主要是打开socket侦听服务，并接收客户端发来的各种请求，通讯细节由Dubbo自己实现。
