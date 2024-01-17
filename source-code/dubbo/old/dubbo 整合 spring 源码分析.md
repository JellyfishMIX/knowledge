# dubbo 整合 spring 源码分析



## 说明

1. 本文基于 jdk 8, dubbo 2.7.18 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



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

这里面初始化了 dubbo 配置文件，可以看到有各种 dubbo xml 配置中可能用到的标签。服务发布相关逻辑在 ServiceBean 中。

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

