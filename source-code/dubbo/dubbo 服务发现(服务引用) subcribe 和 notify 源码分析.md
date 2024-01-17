# dubbo 服务发现 subcribe 和 notify 源码分析



## 说明

1. 本文基于 netty 2.6.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 服务发现相关内容

服务发现相关内容包括: 触发服务发现流程，订阅服务变更事件，获取最新服务信息和生成服务接口代理。



## 触发服务发现流程

com.alibaba.dubbo.config.spring.ReferenceBean

```
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
}
```

ReferenceBean 接口的声明，实现的接口有以下功能:

1. FactoryBean: spring-ioc 创建当前 bean 时，得到的不是 bean xml 配置指定类的实例，而是该 FactoryBean 的 getObject 方法所返回的对象。
   1. 如果 bean 配置未指定构造函数，spring 通过反射机制创建 bean 标签的 class 属性指定实例，然后根据 bean 标签中的配置填充实例的属性。某些 bean 的实例化过程可能比较复杂，需要复杂的配置，配置灵活性受影响。对于这种 bean，spring 提供了 FactoryBean 机制，简单配置一个 FactoryBean，由 FactoryBean 负责真正实例对象复杂的实例化逻辑。
2. ApplicationContextAware: 让当前 bean 可以感知获取 spring 的 ApplicationContext 中其他 bean。
3. InitializingBean: 提供一个生命周期方法的调用时机。spring-ioc 创建当前 bean 时，bean 属性填充后，会调用 InitializingBean#afterPropertiesSet 方法。
4. DisposableBean: 提供一个生命周期方法的调用时机。允许在 spring-ioc 容器销毁该 bean 时触发一次回调。

### 引用对象实例化时机1 -- InitializingBean#afterPropertiesSet

com.alibaba.dubbo.config.spring.ReferenceBean#afterPropertiesSet

1. 当 ReferenceBean 初始化时，afterPropertiesSet 生命周期方法提供一次实例化引用对象的时机。
2. 判断是否需要在创建 ReferenceBean 时就初始化引用对象。dubbo:reference 或 dubbo:consumer 的属性 init 需要为 true，默认为 true。

```java
	@Override
    @SuppressWarnings({"unchecked"})
    public void afterPropertiesSet() throws Exception {
        // ...
        
        // 判断是否需要在创建 ReferenceBean 时就初始化引用对象。<dubbo:reference/>或<dubbo:consumer/> 的属性 init 需要为 true，默认为 true
        Boolean b = isInit();
        if (b == null && getConsumer() != null) {
            b = getConsumer().isInit();
        }
        if (b != null && b.booleanValue()) {
            getObject();
        }
    }
```

### 引用对象实例化时机2 -- ReferenceBean#getObject

ReferenceBean 是一个 FactoryBean，而不是真正供服务引用方使用的实例对象。对于一个 FactoryBean，最重要的就是它的 getObject 方法，创建实例对象的逻辑在于此。

注: 通过注解注入一个 bean 实例，如果 bean 实例在 ioc 三级缓存中不存在，会触发 bean 实例对应的 FactoryBean 执行 getObject 方法，来获取到 bean 实例，这个触发过程不在本文分析范围内。

```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
	@Override
    public Object getObject() throws Exception {
        return get();
    }
}

public class ReferenceConfig<T> extends AbstractReferenceConfig {
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
}
```

### ReferenceConfig#init -- 初始化引用对象实例 ref

com.alibaba.dubbo.config.ReferenceConfig#init

```java
    private void init() {
        if (initialized) {
            return;
        }
        initialized = true;
        // ...

        // 核心逻辑，创建 proxy
        ref = createProxy(map);
        
        // ...
    }
```

### ReferenceConfig#createProxy -- 创建 Invoker 代理

com.alibaba.dubbo.config.ReferenceConfig#createProxy

1. 单点直连一套模式，注册中心一套模式。
2. 注册中心模式，注册中心配置里，组装 URL。
3. 遍历所有的 URL，创建多个 Invoker。
4. 通过 Cluster 将多个 Invoker 合并成一个 Invoker，内部通过策略调用具体的 Invoker。
5. 创建服务代理，在调用服务时，调用的接口实现，就是这个代理对象，此代理对象:
      1. 代理了接口的所有方法。
      2. 封装了 InvocationHandler，InvocationHandler 里封装了 Invoker，而 Invoker 里封装了调用远程服务的逻辑。

```java
    @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        // ...

        // 是否应该做本地引用，默认不做本地引用
        if (isJvmRefer) {
            URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
            invoker = refprotocol.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            // 通过配置 <dubbo:reference/> 的 url 参数，实现点对点的通讯处理逻辑
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                // ...
            } else { // assemble URL from register center's configuration
                // 注册中心模式，组装 URL
                List<URL> us = loadRegistries(false);
                if (us != null && !us.isEmpty()) {
                    // 加载所有注册中心，并转成 URL 集合
                    for (URL u : us) {
                        URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        // 给每个注册中心 URL 添加 refer 参数
                        // refer 参数的值，就是服务消费者的基本信息，包括应用名, 版本号, 引用的接口等
                        urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                // ...
            }

            // 只有一个 url，可能是单个注册中心，或点对点的服务直连
            if (urls.size() == 1) {
                invoker = refprotocol.refer(interfaceClass, urls.get(0));
            } else {
                // 多个 url，可能是是多注册中心，或多服务提供者
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                // 遍历所有的 url，创建多个 Invoker
                for (URL url : urls) {
                    invokers.add(refprotocol.refer(interfaceClass, url));
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                // 通过 Cluster 将多个 Invoker 合并成一个 Invoker，内部通过策略调用具体的 Invoker
                if (registryURL != null) { // registry url is available
                    // use AvailableCluster only when register's cluster is available
                    URL u = registryURL.addParameterIfAbsent(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                    invoker = cluster.join(new StaticDirectory(u, invokers));
                } else { // not a registry url
                    invoker = cluster.join(new StaticDirectory(invokers));
                }
            }
        }

        // ...

        // invoker 不可用时，抛异常
        if (c && !invoker.isAvailable()) {
            // ...
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + serviceKey + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        
        /*
         * create service proxy
         * 创建服务代理，在调用服务时，调用的接口实现，就是这个代理对象，此代理对象:
         * 1. 代理了接口的所有方法
         * 2. 封装了 InvocationHandler，InvocationHandler 里封装了 Invoker，而 Invoker 里封装了调用远程服务的逻辑
         */
        return (T) proxyFactory.getProxy(invoker);
    }
```



## 订阅服务变更事件，监听服务变更

### RegistryProtocol#refer

com.alibaba.dubbo.registry.integration.RegistryProtocol#refer

```java
    @Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        url = url.setProtocol(url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)).removeParameter(Constants.REGISTRY_KEY);
        Registry registry = registryFactory.getRegistry(url);
        // ...

        /*
         * 调用 doRefer，完成 Invoker 的创建和注册中心的订阅
         * 返回的是 MockClusterWrapper
         */
        return doRefer(cluster, registry, type, url);
    }
```

### RegistryProtocol#doRefer

com.alibaba.dubbo.registry.integration.RegistryProtocol#doRefer

```java
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // RegistryDirectory 实现了 NotifyListener 接口，订阅了注册中心的服务信息变更事件，当发生变更后刷新服务目录
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
                && url.getParameter(Constants.REGISTER_KEY, true)) {
            URL registeredConsumerUrl = getRegisteredConsumerUrl(subscribeUrl, url);
            registry.register(registeredConsumerUrl);
            directory.setRegisteredConsumerUrl(registeredConsumerUrl);
        }
        // 注册服务变更事件
        directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
                Constants.PROVIDERS_CATEGORY
                        + "," + Constants.CONFIGURATORS_CATEGORY
                        + "," + Constants.ROUTERS_CATEGORY));

        // 将 RegistryDirectory，合并为一个虚拟的 Invoker
        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```

### RegistryDirectory#subscribe -- 服务订阅

com.alibaba.dubbo.registry.integration.RegistryDirectory#subscribe

```java
    public void subscribe(URL url) {
        setConsumerUrl(url);
        // 对于 zookeeper 会调用 ZookeeperRegistry#doSubScribe
        registry.subscribe(url, this);
    }
```

### ZookeeperRegistry#doSubscribe -- 服务订阅

com.alibaba.dubbo.registry.zookeeper.ZookeeperRegistry#doSubscribe

```java
    @Override
    protected void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            // 服务接口为 * 的订阅逻辑
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                // ...
                // 默认情况下，执行的是下面 else 的订阅逻辑
            } else {
                // 订阅服务信息变更事件
                List<URL> urls = new ArrayList<URL>();
                // 给[/dubbo/服务名]下的节点，添加监听
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        // 创建一个 ChildListener，重写 childChanged 方法调用 notify() 方法
                        listeners.putIfAbsent(listener, new ChildListener() {
                            @Override
                            public void childChanged(String parentPath, List<String> currentChilds) {
                                ZookeeperRegistry.this.(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                            }
                        });
                        zkListener = listeners.get(listener);
                    }
                    // 创建 path
                    zkClient.create(path, false);
                    // 通过 zkClient，给 path 添加 Listener
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                // 订阅时，直接调用 notify() 通知一次，完成 invoker 的首次创建
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```



## 获取最新服务信息，刷新服务信息

有两个地方调用 notify 方法:

1. 第一次订阅的时候。
2. 接收到服务变更事件后。

notify 方法最终会调用 RegistryDirectory#refreshInvoker

### RegistryDirectory#refreshInvoker -- 刷新 RegistryDirectory 中的 Invoker

com.alibaba.dubbo.registry.integration.RegistryDirectory#refreshInvoker

1. 刷新 Invoker 分三个步骤，创建，保存和销毁。
   1. 创建 Invoker，将消费者和服务端通讯的逻辑封装成 Invoker。
   2. 保存 Invoker，将创建好的 Invoker 保存到 RegistryDirectory 持有的引用对象中。
   3. 销毁 Invoker，将没用的 Invoker 都销毁掉。

```java
    private void refreshInvoker(List<URL> invokerUrls) {
        // 处理所有服务提供者都停止的情况
        if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
                && Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            this.forbidden = true; // Forbid to access
            this.methodInvokerMap = null; // Set the method invoker map to null
            destroyAllInvokers(); // Close all invokers
        } else {
            // ...
            // 将 invokerUrls，转化为 Map<Invoker>，内含生成 Invoker 的逻辑
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map
            Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap); // Change method name to map Invoker Map
            // state change
            // If the calculation is wrong, it is not processed.
            if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
                return;
            }
            // 保存 invoker
            this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
            this.urlInvokerMap = newUrlInvokerMap;
            try {
                // 销毁没用的 invoker
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```

### 创建 DubboInvoker

```java
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);

    // 创建 DubboInvoker
    // 其核心的连接服务端的实现，在 getClients 里
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);

    return invoker;
}
```



## 创建服务接口代理，代理接口的调用

在 consumer 视角看来，dubbo 服务是接口，接口的实现是 dubbo 生成的代理对象，consumer 调用服务接口使用的是代理对象。创建服务代理的方法如下

ReferenceConfig#createProxy

```java
private T createProxy(Map<String, String> map) {
    // 省略创建 Invoker 的代码，前面已分析...
    
    /* 
     * 创建服务代理
     * PROXY_FACTORY 是 ProxyFactory 的自适应类 ProxyFactory&Adaptive, 默认为 JavassistProxyFactory
     * 最终，调用的是 JavassistProxyFactory.getProxy()
     * 
     * 这个代理对象：
     * 1. 生成了接口方法的所有代理方；
     * 2. 封装了 InvocationHandler，InvocationHandler里封装了invoker，而invoker里面封装了调用远程服务的逻辑。
     * 服务调用的大致链路:
     * consumer.proxy-> InvocationHandler -> invoker ->
     * ExchangeClient -> ExchangeChannel -> NettyChannel -> NioSocketChannel -> provider
     */
    return (T) PROXY_FACTORY.getProxy(invoker);
}
```

调用链路例如:

DemoService#sayHello()

```
proxy.sayHello()
	--> InvokerInvocationHandler.invoke()
		--> MockClusterInvoker.invoke()
```

