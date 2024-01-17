# dubbo 服务注册源码分析



## 说明

1. 本文基于 netty 2.6.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 服务注册相关内容

服务注册相关内容包括:

1. 服务的解析: dubbo 服务配置的解析过程。
2. 服务的启动: 以默认的 dubbo 协议，分析服务的启动过程。
3. 服务的注册: 以默认的 dubbo 协议，分析服务的注册过程。
4. 服务信息的存储形式: 简单看下，服务信息在注册中心 zookeeper 中的存储形式。



## 服务的解析

服务注册的第一步是把配置信息解析成服务信息。

以 spring 集成 dubbo 举例，对外暴露的服务是在 xml 文件中通过 `<dubbo:service>` 配置。服务的解析就是将配置的 `<dubbo:service>` 标签，解析为 `ServiceBean`。

dubbo 的 xml 配置是通过 spring 的自定义标签功能实现的，服务的解析过程就是 spring 自定义标签的解析过程。

1. 编写 dubbo.xsd 文件，定义 dubbo 配置的 xml 规范。
2. 编写 spring.schemas 文件。
3. 编写 spring.handlers 文件。
4. 自定义 `DubboNamespaceHandler`，继承自 `NamespaceHandlerSupport`，将 `<dubbo:service>` 解析为 `ServiceBean`。

### 编写 dubbo.xsd 文件

dubbo.xsd 文件中定义了 dubbo 配置的 xml 规范，规定有 dubbo 哪些标签，可以配置哪些属性，属性配置的约束等。里面规范太多，以 service 标签的定义举例。

标签名为 service，类型是 serviceType，这个类型也需要定义。

```xml
<!-- 标签名为 service，类型是 serviceType -->
<xsd:element name="service" type="serviceType">
    <xsd:annotation>
        <xsd:documentation><![CDATA[ Export service config ]]></xsd:documentation>
        <xsd:appinfo>
            <tool:annotation>
                <tool:exports type="org.apache.dubbo.config.ServiceConfig"/>
            </tool:annotation>
        </xsd:appinfo>
    </xsd:annotation>
</xsd:element>
```

对 serviceType 类型的定义。

```xml
<!-- 对serviceType的详细定义 -->
<xsd:complexType name="serviceType">
    <xsd:complexContent>
        <xsd:extension base="abstractServiceType">
            <xsd:choice minOccurs="0" maxOccurs="unbounded">
                <xsd:element ref="method" minOccurs="0" maxOccurs="unbounded"/>
                <xsd:element ref="parameter" minOccurs="0" maxOccurs="unbounded"/>
                <xsd:element ref="beans:property" minOccurs="0" maxOccurs="unbounded"/>
            </xsd:choice>
            <!-- interface 属性 -->
            <xsd:attribute name="interface" type="xsd:token" use="required">
                <xsd:annotation>
                    <xsd:documentation>
                        <![CDATA[ Defines the interface to advertise for this service in the service registry. ]]></xsd:documentation>
                    <xsd:appinfo>
                        <tool:annotation>
                            <tool:expected-type type="java.lang.Class"/>
                        </tool:annotation>
                    </xsd:appinfo>
                </xsd:annotation>
            </xsd:attribute>
            <!-- ref 属性 -->
            <xsd:attribute name="ref" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation>
                        <![CDATA[ The service implementation instance bean id. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <!-- class 属性 -->
            <xsd:attribute name="class" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ The service implementation class name. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <!-- path 属性 -->
            <xsd:attribute name="path" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ The service path. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <!-- provider 属性 -->
            <xsd:attribute name="provider" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ Deprecated. Replace to protocol. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <!-- generic 属性 -->
            <xsd:attribute name="generic" type="xsd:string">
                <xsd:annotation>
                    <xsd:documentation><![CDATA[ Generic service. ]]></xsd:documentation>
                </xsd:annotation>
            </xsd:attribute>
            <xsd:anyAttribute namespace="##other" processContents="lax"/>
        </xsd:extension>
    </xsd:complexContent>
</xsd:complexType>
```

### 编写 spring.schemas

spring.schemas 文件的作用是指定 xsd 文件位置。

```
http://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
http://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/compat/dubbo.xsd
```

声明的 xsd 文件会在 dubbo 配置中引入，schemaLocation 属性，成对出现的两个 url，第一个 url 表示想导入的 xsd 文件的 namespace，第二个 url 定义 xsd 文件的位置:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://dubbo.apache.org/schema/dubbo        http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
</beans>
```

关于 xml schema 更多内容可以看这篇文章: https://blog.csdn.net/weixin_43735348/article/details/127914660

### 编写 spring.handlers

spring.handlers 作用是指定标签解析的实现类。

```
http://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```

使用 DubboNamespaceHandler 解析

### 使用 DubboNamespaceHandler 解析

1. 在 DubboNamespaceHandler 初始化时，注册 DubboBeanDefinitionParser。
2. 在 DubboBeanDefinitionParser 中完成解析逻辑。

com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler#init

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        // 注册各标签的 BeanDefinitionParser
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```

#### DubboBeanDefinitionParser

com.alibaba.dubbo.config.spring.schema.DubboBeanDefinitionParser#parse(org.w3c.dom.Element, ParserContext, java.lang.Class<?>, boolean)

```java
    /**
     * 解析各种类型的 bean 的方法，这里只保留了解析 ServiceBean 的逻辑
     */
    private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
        String id = element.getAttribute("id");
        //...
        // 解析 ServiceBean
		if (ServiceBean.class.equals(beanClass)) {
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
        }
        // ...
        return beanDefinition;
    }
```



## 服务的启动

服务启动指的是，启动包括第三方依赖在内的组件，达到可服务状态。如启动 netty 或 MINA 提供网络 IO 能力。

服务注册也叫服务暴露，指的是将启动后服务的信息，登记到注册中心暴露出去，这样调用方就可以从注册中心获取服务信息了。

### ServiceBean -- 监听 ContextRefreshedEvent 事件触发服务的启动和注册。

1. ServiceBean 监听了 ContextRefreshedEvent 事件，spring-ioc 初始化时，会发布 ContextRefreshedEvent 事件，调用 onApplicationEvent 方法。
2. dubbo 服务的启动和注册，从 onApplicationEvent 方法开始。

```java
/**
 * ServiceBean 监听了 ContextRefreshedEvent 事件，spring-ioc 初始化时，会发布 ContextRefreshedEvent 事件，调用 onApplicationEvent 方法。
 * dubbo 服务的启动和注册，从 onApplicationEvent 方法开始
 */
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,
        ApplicationEventPublisherAware {
}
```

com.alibaba.dubbo.config.spring.ServiceBean#onApplicationEvent

1. 调用 ServiceBean#export 方法，执行服务暴露(启动和注册)核心方法。

```java
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (isDelay() && !isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            // 服务暴露(启动和注册)核心方法
            export();
        }
    }
```

### ServiceConfig#export 方法 -- 执行服务暴露(启动和注册)

com.alibaba.dubbo.config.spring.ServiceBean#export

1. 调用父类 ServiceConfig#export 方法，延迟暴露或立即暴露。延迟暴露加了定时调度进行暴露，调用的都是 doExport 方法。
2. 发布 ServiceBeanExportedEvent 事件，可以通过订阅事件，来定制服务暴露后的业务处理逻辑。

```java
    @Override
    public void export() {
        super.export();
        // Publish ServiceBeanExportedEvent
        // 发布 ServiceBeanExportedEvent 事件，可以通过订阅事件，来定制服务暴露后的业务处理逻辑
        publishExportEvent();
    }

	/**
	 * com.alibaba.dubbo.config.ServiceConfig#export, 即上面的 super.export
	 */
    public synchronized void export() {
        // 检查必要的配置
        if (provider != null) {
            if (export == null) {
                export = provider.getExport();
            }
            if (delay == null) {
                delay = provider.getDelay();
            }
        }
        if (export != null && !export) {
            return;
        }

        // 延迟暴露
        if (delay != null && delay > 0) {
            delayExportExecutor.schedule(new Runnable() {
                @Override
                public void run() {
                    doExport();
                }
            }, delay, TimeUnit.MILLISECONDS);
        } else {
            // 立即暴露
            doExport();
        }
    }
```

### ServiceConfig#doExport -- 执行服务暴露

com.alibaba.dubbo.config.ServiceConfig#doExport

1. 核心调用了 doExportUrls 方法，执行服务暴露。

```java
protected synchronized void doExport() {
    //...
    
    // 执行启动和暴露服务
    doExportUrls();
}
```

### ServiceConfig#doExportUrls -- 执行服务的暴露

com.alibaba.dubbo.config.ServiceConfig#doExportUrls

1. 加载注册中心，后面要将服务注册到所有的注册中心中。
2. protocols 为要暴露的协议，遍历每个协议分别执行一次暴露。

```java
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void doExportUrls() {
        // 加载注册中心，后面要将服务注册到所有的注册中心中
        // registryURLs 示例: registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=provider-app&dubbo=2.0.2&pid=78430&registry=zookeeper&release=2.7.4.1&timestamp=1644651852219
        List<URL> registryURLs = loadRegistries(true);
        // protocols 为要注册的协议，遍历每个协议分别执行一次启动和暴露。示例：<dubbo:protocol name="dubbo" port="20880" valid="true" id="dubbo" prefix="dubbo.protocols." />
        for (ProtocolConfig protocolConfig : protocols) {
            // 执行服务的暴露
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```

### dubbo 中的 URL

dubbo 中 URL 可以看做是一个封装的信息承载结构，用于信息的传递。可以简单的理解为 dubbo 中的 URL 是一个承载信息的实体类，在整个流程中负责数据传递(主要用于在各个 extension 之间传递数据)。

dubbo URL 格式，可以包含如下的几个部分，某些部分可缺省

```
protocol://username:password@host:port/path?key=value&key=value
```

1. protocol：一般是 dubbo 中的各种协议 如: dubbo thrift http zk

2. username/password: 用户名/密码

3. host/port: 主机/端口

4. path: 接口名称

5. parameters(key-value): 参数键值对

```
# 描述一个 dubbo 协议的服务
dubbo://192.168.1.6:20880/moe.cnkirito.sample.HelloService?timeout=3000

# 描述一个 zookeeper 注册中心
zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.2&interface=org.apache.dubbo.registry.RegistryService&pid=1214&qos.port=33333&timestamp=1545721981946

# 描述一个消费者
consumer://30.5.120.217/org.apache.dubbo.demo.DemoService?application=demo-consumer&category=consumers&check=false&dubbo=2.0.2&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=1209&qos.port=33333&side=consumer&timestamp=1545721827784
```

更详细的参考 link：[Dubbo 中的 URL 统一模型](https://zhuanlan.zhihu.com/p/98561575)

### ServiceConfig#doExportUrlsFor1Protocol -- 执行一个 protocol 的暴露

com.alibaba.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol

1. 协议名，默认为 dubbo。
2. 初始化服务信息。比如应用名, 接口名, 方法名等。
3. 将服务信息转换成 url。
4. 如果配置不是 remote，则向本地暴露服务，如果配置不是 local，则向远程暴露服务。
5. 暴露服务，最终会调用 DubboProtocol#export 启动服务, Registry#register 注册服务。

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        // 协议名，默认为 dubbo
        String name = protocolConfig.getName();
        if (name == null || name.length() == 0) {
            name = "dubbo";
        }

        // 下面一大段代码，初始化服务信息。比如应用名, 接口名, 方法名等
        Map<String, String> map = new HashMap<String, String>();
        map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
        map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    	// ...

        // 将服务信息转换成 url
        String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
        URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);

        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(Constants.SCOPE_KEY);
        // don't export when none is configured
        if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
            // 如果配置不是 remote，则向本地暴露服务
            if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            // 如果配置不是 local，则向远程暴露服务
            if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (registryURLs != null && !registryURLs.isEmpty()) {
                    for (URL registryURL : registryURLs) {
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        // 暴露服务，协议为 dubbo 会调用 DubboProtocol#export 启动服务，协议为 registry 会调用 Registry#register 注册服务。
                        // 一般 Serviceconfig 会配置 dubbo 协议和 registry 协议，所以两个协议都会调用。
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    // ...
                }
                // ...
            }
        }
        this.urls.add(url);
    }
```

服务的暴露分为本地和远程，对应 exportLocal(url) 和 protocol.export(wrapperInvoker)。本地暴露不做展开，主要说远程暴露。

### RegistryProtocol#protocol -- 自适应类

RegistryProtocol#protocol 是自适应类 Protocol$Adaptive。远程暴露时(即遍历到的 protocol 为 registry)，url 里面的协议参数是 registry，实际会调用 RegistryProtocol.export()。

```java
// RegistryProtocol 中的 protocol 定义，是 Protocol 的自适应类 Protocol$Adaptive
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

### RegistryProtocol#export 服务暴露

com.alibaba.dubbo.registry.integration.RegistryProtocol#export

1. 导出 invoker，内部会获取 providerUrl。然后通过 Protocol$Adaptive 调用 Protocol 的 Extension。因为 providerUrl 里的 protocol 已经变为了 dubbo，所以实际上调用了 DubboProtocol#export。在 DubboProtocol#export 中，实现了服务的启动。
2. 获取注册中心 URL, URL 中的 protocol=registry。将 protocol 替换成具体注册中心的协议，如 zookeeper。
3. 获取服务提供者 URL。与 registryUrl 相比，protocol 变成了 dubbo。
4. 调用服务注册核心方法 RegistryProtocol#register。
5. 综上所述，暴露服务，最终会调用 DubboProtocol#export 启动服务, Registry#register 注册服务。

```java
    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //export invoker
        // 导出 invoker，内部会获取 providerUrl。然后通过 Protocol$Adaptive 调用 Protocol 的 Extension。
        // 因为 providerUrl 里的 protocol 已经变为了 dubbo，所以实际上调用了 DubboProtocol#export。在 DubboProtocol#export 中，实现了服务的启动。
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);

        // 注册中心 URL, URL 中的 protocol=registry。将 protocol 替换成具体注册中心的协议，如 zookeeper
        URL registryUrl = getRegistryUrl(originInvoker);

        //registry provider
        final Registry registry = getRegistry(originInvoker);
        // 服务提供者 URL。与 registryUrl 相比，protocol 变成了 dubbo
        final URL registeredProviderUrl = getRegisteredProviderUrl(originInvoker);

        //to judge to delay publish whether or not
        boolean register = registeredProviderUrl.getParameter("register", true);

        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registeredProviderUrl);

        if (register) {
            // 服务注册核心方法
            register(registryUrl, registeredProviderUrl);
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
        }

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registeredProviderUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registeredProviderUrl);
    }
```

### DubboProtocol#export -- 服务启动

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#export

服务启动，封装 Invoker 得到 Exporter，启动用于网络 IO 的 exchangeServer。

1. 创建一个 DubboExporter，封装 Invoker。
2. 打开 exchangeServer 监听服务，用于网络 IO。

```java
    /**
     * 服务导出功能，根据本地执行的 Invoker 得到 Exporter
     */
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.
        String key = serviceKey(url);
        // 创建一个 DubboExporter，封装 Invoker。
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        // 启动 exchangeServer 监听服务，用于网络 IO
        openServer(url);

        return exporter;
    }
```

### DubboProtocol#openServer -- 启动网络 IO 组件 ExchangeServer

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#openServer

启动 exchangeServer 监听服务。exchangeServer 是负责网络 IO 的组件。后面 client 端向 server 端发送信息时，通过通信对象 exchangeServer 获取客户端传来的 Invocation 参数。

1. invoker 的 url 的 address 用来标识 exchangeServer
2. 尝试在 exchangeServerMap 中获取目标 exchangeServer
   1. 如果目标 exchangeServer 还没有，则创建并加入 exchangeServerMap 中。
   2. 如果目标 exchangeServer 已有，则重置其关联的 url。

```java
    /**
     * 启动 exchangeServer 监听服务。exchangeServer 是负责网络 IO 的组件。
     * 后面 client 端向 server 端发送信息时，通过通信对象 exchangeServer 获取客户端传来的 Invocation 参数。
     *
     * @param url 本地执行的 invoker 的 url
     */
    private void openServer(URL url) {
        // find server.
        // invoker 的 url 的 address 用来标识 exchangeServer
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
        if (isServer) {
            // 尝试在 exchangeServerMap 中获取目标 exchangeServer
            ExchangeServer server = serverMap.get(key);
            // 如果目标 exchangeServer 还没有，则创建并加入 exchangeServerMap 中
            if (server == null) {
                serverMap.put(key, createServer(url));
            } else {
                // server supports reset, use together with override
                // 如果目标 exchangeServer 已有，则重置其关联的 url
                server.reset(url);
            }
        }
    }
```

### DubboProtocol#createServer

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#createServer

调用 Exchangers#bind 方法, 启动 ExchangeServer。

```java
    private ExchangeServer createServer(URL url) {
        // ...
        
        ExchangeServer server;
        try {
            // 启动 ExchangeServer 的核心方法
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        // ...
        return server;
    }
```

### Exchangers#bind -- 启动 ExchangeServer 的核心方法

com.alibaba.dubbo.remoting.exchange.Exchangers#bind(URL, com.alibaba.dubbo.remoting.exchange.ExchangeHandler)

1. 获取 Exchanger, 默认实现是 HeaderExchanger，然后调用 Exchanger#bind 方法启动 ExchangeServer。

```java
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        // ...
        // 获取 Exchanger, 默认实现是 HeaderExchanger，然后调用 Exchanger#bind 方法启动 ExchangeServer
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).bind(url, handler);
    }

    public static Exchanger getExchanger(URL url) {
		// 默认的 type 是 header, 即 Exchanger 默认的 extension 是 HeaderExchanger
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        return getExchanger(type);
    }

    public static Exchanger getExchanger(String type) {
        // 根据 type 获取具体的 extension
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
    }
```

### HeaderExchanger#bind -- 启动 ExchangeServer

com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchanger#bind

调用核心方法 Transporters#bind 启动网络 IO 组件

```java
public class HeaderExchanger implements Exchanger {
	@Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        // 调用核心方法 Transporters#bind 启动网络 IO 组件
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
}
```

### Transporters#bind -- 启动具体的网络 IO 组件

com.alibaba.dubbo.remoting.Transporters#bind(URL, com.alibaba.dubbo.remoting.ChannelHandler...)

1. 通过 ExtensionLoader 获取 Transporter 的 adaptiveExtension(即具体的网络 IO 组件)，默认为 NettyTransporter，另外还有 MinaTransporter, GrizzlyTransporter。

```java
    public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        // ...
        
        // 通过 ExtensionLoader 获取 Transporter 的 adaptiveExtension(即具体的网络 IO 组件)，默认为 NettyTransporter，另外还有 MinaTransporter, GrizzlyTransporter
        return getTransporter().bind(url, handler);
    }

	public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
```

### NettyTransporter#bind -- 创建一个 NettyServer 实例

com.alibaba.dubbo.remoting.transport.netty4.NettyTransporter#bind

```java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty";

    @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
    
    // ...
}
```

### NettyServer#doOpen -- 启动 nettyServer

com.alibaba.dubbo.remoting.transport.netty4.NettyServer#doOpen

1. 父类 AbstractServer 的构造方法使用了模板方法模式，调用了此子类的 doOpen() 方法

```java
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
    	// 父类 AbstractServer 的构造方法使用了模板方法模式，调用了此子类的 doOpen() 方法
    	super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
	}

    @Override
    protected void doOpen() throws Throwable {
        // netty server
        bootstrap = new ServerBootstrap();

        // netty parent 线程组
        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        // netty child 线程组
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        // 处理 netty 消息的 handler
        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        // 配置 bootstrap
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        // 绑定端口并启动服务
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();
    }
```



## 服务的注册

### RegistryProtocol#register 向注册中心注册

com.alibaba.dubbo.registry.integration.RegistryProtocol#register

使用模版方法模式，根据 registryUrl，先获取 Registry 实例，再调用 Registry#register 进行注册。

```java
    public void register(URL registryUrl, URL registedProviderUrl) {
        // 获取注册中心对象 Registry，这里的 registryFactory，是自适应扩展 RegistryFactory@Adaptive
        // 对于 registryUrl.protocol=zookeeper，获取到的是 ZookeeperRegistryFactory
        // 最后通过 ZookeeperRegistryFactory#createRegistry 创建并获取注册中心对象，ZookeeperRegistry
        Registry registry = registryFactory.getRegistry(registryUrl);
        // 使用了模板方法设计模式，通过 FailBackRegistry，调用具体实现类的 doRegister 方法。如 ZookeeperRegister#doRegister
        // 最终通过 ZookeeperRegister#doRegister，注册服务信息，将服务信息写入注册中心。
        registry.register(registedProviderUrl);
    }
```

### ZookeeperRegistryFactory#createRegistry

创建一个 ZookeeperRegistry 实例并返回。

```java
    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
```

### FailbackRegistry#register

ZookeeperRegistry 的 register 方法，继承自父类 FailbackRegistry，调用 ZookeeperRegistry 实现的 doRegister 方法。

```java
public class ZookeeperRegistry extends FailbackRegistry
```

```java
    @Override
    public void register(URL url) {
        // ...
		doRegister(url);
		// ...
    }
```

### ZookeeperRegistry#doRegister

通过 zk 客户端，将服务信息写入注册中心。

```java
    @Override
    protected void doRegister(URL url) {
        try {
            // 通过 zk 客户端，将服务信息写入注册中心
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```



## 服务信息的存储形式

以 zookeeper 作为注册中心举例，所有信息存储在 zookeeper 的节点上，通过 zookeeper 的 watch 机制，实现服务的订阅通知。

zookeeper 是一个树形结构，跟文件夹目录一样。zookeeper 的每个节点可以是目录的同时，也可以存储数据。

下面是服务注册完成后，在 Zookeeper 上的存储目录：

// dubbo 所有数据都保存在根节点[/dubbo]下

- dubbo
  - com.apache.dubbo.DemoService // 服务名
    - configurators // 配置类信息
    - consumers // 消费者信息
    - providers // 服务提供者信息
    - routers // 路由信息

dubbo 所有数据都保存在根节点[/dubbo]下，以服务名为子目录，在下一级子目录里保持了服务的配置信息、消费者信息、提供者信息和路由信息。