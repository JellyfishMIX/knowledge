# dubbo 服务发布相关源码分析



## 说明

1. 本文基于 jdk 8, dubbo 2.5.x 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## dubbo 服务发布机制概述

![这里写图片描述](https://image-hosting.jellyfishmix.com/20221101013949.jpeg)

1. 首先 ServiceConfig 拿到对外提供服务的实现类 ref(例如: HelloWorldImpl)，然后通过 ProxyFactory#getInvoker 方法使用 ref 生成一个 AbstractProxyInvoker 实例，完成服务实现(ServiceImpl)到 Invoker 的转化。

2. dubbo 服务暴露的关键在 Invoker 转换到 Exporter 的过程(如上图中的红色部分)，DubboProtocol (Protocol 接口的默认实现)的 Invoker 转换为 Exporter 发生在 DubboProtocol 类的 export 方法，此方法中还打开了 ExchangeServer 监听服务，以接收客户端发来的各种请求。

3. 后面 client 端向 server 端发送信息时，通过通信对象 ExchangeServer 获取客户端传来的 Invocation 参数。然后找到对应的 Exporter，Exporter 关联了本地执行的 Invoker，就可以执行服务方法了。



## dubbo 中的 URL

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



## Invoker

com.alibaba.dubbo.rpc.Invoker

Invoker 提供了可调用的能力。调用 Invoker#invoke 方法，传入 invocation 参数。

```java
public interface Invoker<T> {
    Class<T> getInterface();

    URL getUrl();

    Result invoke(Invocation invocation) throws RpcException;

    void destroy();
}
```

Invoker 根据执行过程划分，有三种类型：

1. 本地执行的 Local Invoker: server 端拥有的 invoker，含有 dubbo 接口对应的 ServiceImpl 实现，通过反射执行 ServiceImpl 对应的方法。
2. 远程通信执行的 Remote Invoker: client 端拥有的 invoker，执行该接口的方法，需要进行远程通信，发送要执行的参数信息给 server 端。server 端使用本地执行的 Local Invoker 执行相应的方法，然后将执行结果返回给 client 端。这个过程是经典的 rpc 调用。
3. 多个 Remote Invoker 聚合成的集群版 Invoker: client 端拥有某个服务的多个 Invoker，将多个 Invoker 聚合成一个集群版的 Invoker。client 端使用时，只需通过集群版的 Invoker 来进行操作。集群版的 Invoker 会从众多的远程通信执行的 Invoker 中选择一个来执行(从中加入负载均衡, 服务降级等策略)，类似于服务治理，dubbo 已经实现了。



## Invocation 接口

com.alibaba.dubbo.rpc.Invocation

Invocation 接口，提供了充当 Invoker#invoke 方法参数载体的能力。包含了要执行的方法，参数等信息。目前实现类只有一个 RpcInvocation。

```java
public interface Invocation {
    URL getUrl();

    String getMethodName();

    Class<?>[] getParameterTypes();

    Object[] getArguments();
}
```



## ProxyFactory

com.alibaba.dubbo.rpc.ProxyFactory

ProxyFactory 接口提供代理 invoker 的能力。

1. ProxyFactory#getProxy 方法供 client 端使用，创建出代理对象。
2. ProxyFactory#getInvoker 方法供 server 端使用，将服务对象如 ServiceImpl 包装成一个 Invoker 对象，通过反射来执行具体的 ServiceImpl 对象的方法。
3. ProxyFactory 接口的实现有 JdkProxyFactory, JavassistProxyFactory 等，默认是 JavassistProxyFactory。

```java
@SPI("javassist")
public interface ProxyFactory {
    /**
     * 供 client 端使用，创建出代理对象
     */
    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    /**
     * 供 server 端使用，将服务对象如 ServiceImpl 包装成一个 Invoker 对象，通过反射来执行具体的 ServiceImpl 对象的方法。
     */
    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}
```



## JavassistProxyFactory

com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory

ProxyFactory 接口的默认实现是 JavassistProxyFactory

1. JavassistProxyFactory#getInvoker 方法，创建了一个 AbstractProxyInvoker 实例。这个实例重写了 doInvoke 方法，可以调用 wrapper(具体服务实现 ServiceImpl 的包装对象)。
2. 这个动作让这里创建的 AbstractProxyInvoker 实例，定位是本地执行的 Invoker: server 端拥有的 invoker，含有 dubbo 接口对应的 ServiceImpl 实现，通过反射执行 ServiceImpl 对应的方法。

```java
/**
 * JavaassistRpcProxyFactory
 */
public class JavassistProxyFactory extends AbstractProxyFactory {

    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```



## AbstractProxyInvoker

1. AbstractProxyInvoker 对 invoke() 方法的实现: 应用了模版方法模式，invoke 作为模版方法调用了 doInvoke 方法，doInvoke 方法由具体子类实现。AbstractProxyInvoker 子类对 doInvoke 方法的实现，决定了子类的定位是三种 Invoker 中的哪一种。

2. 回顾一下服务发布的第 2 个过程: 使用 ProxyFactory 将 ServiceImpl 封装成一个本地执行的 Invoker。执行这个服务 -> 执行这个本地的 Invoker -> 调用 AbstractProxyInvoker#invoke(Invocation invocation) 方法，通过反射执行 ServiceImpl。

```java
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```



## 引入 Protocol 之前存在的问题

1. 回顾一下服务发布的第 1, 2 个过程:

   1. 获取注册中心信息 dubbo:registry 和协议信息 dubbo:protocol。


   2. 使用 ProxyFactory 将 ServiceImpl 封装成一个本地执行的 Invoker。执行这个服务 -> 执行这个本地的 Invoker -> 调用 AbstractProxyInvoker#invoke(Invocation invocation) 方法，通过反射执行 ServiceImpl。

2. 现在的问题是: 客户端如何调用服务端的方法，和承载着调用参数的 Invocation 来源问题。

3. 上述问题的解决，在 server 端具体来说是 Protocol 所做的事情: 根据指定的协议向注册中心注册 dubbo Service 服务。当客户端根据协议调用这个服务时，将客户端传递过来的 Invocation 参数交给本地执行的 Invoker。所以 Protocol 会加入远程通信模块，为 server 端提供支持，根据 client 端的请求来获取参数 Invocation。



## Protocol

com.alibaba.dubbo.rpc.Protocol

协议接口，实现了此接口的类拥有支持远程通信的能力。

1. Protocol#export 方法: 供 server 端使用，将本地执行的 Invoker 通过协议暴漏给外部(client 端)，得到一个 Exporter。这样外部(client 端)就可以通过协议发送执行参数 Invocation 给 server 端 Exporter，然后 Exporter 交给本地执行的 Invoker 来执行。
2. Protocol#refer 方法: 供 client 端使用，client 端从注册中心获取 server 端发布的服务信息，得到一个远程执行的 Invoker。

```java
/**
 * Protocol. (API/SPI, Singleton, ThreadSafe)
 *
 * 协议接口，实现了此接口的类拥有支持远程通信的能力
 */
@SPI("dubbo")
public interface Protocol {

    /**
     * Get default port when user doesn't config the port.
     *
     * @return default port
     */
    int getDefaultPort();

    /**
     * Export service for remote invocation: <br>
     * 1. Protocol should record request source address after receive a request:
     * RpcContext.getContext().setRemoteAddress();<br>
     * 2. export() must be idempotent, that is, there's no difference between invoking once and invoking twice when
     * export the same URL<br>
     * 3. Invoker instance is passed in by the framework, protocol needs not to care <br>
     *
     * 供 server 端使用，将本地执行的 Invoker 通过协议暴漏给外部(client 端)。这样外部(client 端)就可以通过协议发送执行参数 Invocation，然后交给本地执行的 Invoker 来执行
     *
     * @param <T>     Service type
     * @param invoker Service invoker
     * @return exporter reference for exported service, useful for unexport the service later
     * @throws RpcException thrown when error occurs during export the service, for example: port is occupied
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * Refer a remote service: <br>
     * 1. When user calls `invoke()` method of `Invoker` object which's returned from `refer()` call, the protocol
     * needs to correspondingly execute `invoke()` method of `Invoker` object <br>
     * 2. It's protocol's responsibility to implement `Invoker` which's returned from `refer()`. Generally speaking,
     * protocol sends remote request in the `Invoker` implementation. <br>
     * 3. When there's check=false set in URL, the implementation must not throw exception but try to recover when
     * connection fails.
     *
     * 供 client 端使用，client 端从注册中心获取 server 端发布的服务信息
     *
     * @param <T>  Service type
     * @param type Service class
     * @param url  URL address for the remote service
     * @return invoker service's local proxy
     * @throws RpcException when there's any error while connecting to the service provider
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * Destroy protocol: <br>
     * 1. Cancel all services this protocol exports and refers <br>
     * 2. Release all occupied resources, for example: connection, port, etc. <br>
     * 3. Protocol can continue to export and refer new service even after it's destroyed.
     */
    void destroy();

}
```



## dubbo 服务发布的第 3 个过程

server 端调用 Protocol#export 方法，根据本地执行的 invoker 得到一个 exporter。

```java
Exporter<?> exporter = protocol.export(invoker);
```

protocol 的来源:

```java
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

protocol 是一个 AdaptiveExtension，可以动态地选择不同的 Extension 实现。

```java
public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg0 == null)  { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null"); 
    }
    if (arg0.getUrl() == null) { 
        throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg0.getUrl();
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.export(arg0);
}

public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0,com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException{
    if (arg1 == null)  { 
        throw new IllegalArgumentException("url == null"); 
    }
    com.alibaba.dubbo.common.URL url = arg1;
    String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
    if(extName == null) {
        throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])"); 
    }
    com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)com.alibaba.dubbo.common.ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
    return extension.refer(arg0, arg1);
}
```

这个 AdaptiveExtension 的 export(Invoker invoker) 方法，可以根据 Invoker 中 url 的信息来最终选择 Protocol 的实现，默认实现是 DubboProtocol，然后再对 DubboProtocol 进行依赖注入，wrap 包装 (getExtension()方法)。



## Protocol 的实现

![这里写图片描述](https://image-hosting.jellyfishmix.com/20221106132056.png)

1. 在返回 Protocol 的默认实现 DubboProtocol 之前，经过了 ProtocolFilterWrapper, ProtocolListenerWrapper, RegistryProtocol 的包装。

2. 所谓包装在做如下操作，包装的动作使用装饰器模式，做了类似 AOP 的功能。

```java
package com.alibaba.xxx;

import com.alibaba.dubbo.rpc.Protocol;

public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;

    public XxxProtocol(Protocol protocol) { impl = protocol; }

    // 接口方法做一个操作后，再调用extension的方法
    public Exporter<T> export(final Invoker<T> invoker) {
        //... 一些操作
        impl .export(invoker);
        // ... 一些操作
    }

    // ...
}
```

3. 主要分析默认实现类 DubboProtocol, 包装类 RegistryProtocol，先暂时忽略 ProtocolFilterWrapper, ProtocolListenerWrapper 包装类。
4. 包装类 RegistryProtocol#export 方法，将服务注册到注册中心。



## DubboProtocol#export 方法

服务导出功能，根据本地执行的 Invoker 得到 Exporter。

1. 创建一个 DubboExporter，封装 Invoker。
2. 打开 exchangeServer 监听服务。

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

        // 打开 exchangeServer 监听服务
        openServer(url);

        return exporter;
    }
```



## DubboProtocol#openServer 方法

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#openServer

打开 exchangeServer 监听服务。ExchangeServer 是负责与客户的通信的模块。

后面 client 端向 server 端发送信息时，通过通信对象 ExchangeServer 获取客户端传来的 Invocation 参数。

1. 本地执行的 invoker 的 url 的 address 用来标识 exchangeServer。
2. 尝试在 exchangeServerMap 中获取目标 exchangeServer。
   1. 如果目标 exchangeServer 还没有，则创建并加入 exchangeServerMap 中。
   2. 如果目标 exchangeServer 已有，则重置其关联的 url。

```java
    /**
     * 打开 exchangeServer 监听服务。exchangeServer 是负责与客户的通信的模块。
     * 后面 client 端向 server 端发送信息时，通过通信对象 exchangeServer 获取客户端传来的 Invocation 参数。
     *
     * @param url 本地执行的 invoker 的 url
     */
    private void openServer(URL url) {
        // find server.
        // 本地执行的 invoker 的 url 的 address 用来标识 exchangeServer
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
        if (isServer) {
            // 尝试在 exchangeServerMap 中获取目标 exchangeServer
            ExchangeServer server = serverMap.get(key);
            // 如果目标 exchangeServer 还没有，则创建并加入 exchangeServerMap 中
            if (server == null) {
                serverMap.put(key, createServer(url));
                // 如果目标 exchangeServer 已有，则重置其关联的 url
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
    }
```



## DubboProtocol#requestHandler

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#requestHandler

在 DubboProtocol 中，每个 ExchangeServer 通信对象都绑定了一个 ExchangeHandler 对象，内容如下:

```java
public class DubboProtocol extends AbstractProtocol {
    	// ...
		private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                // 调用 getInvoker 方法来获取本地执行的 invoker(先从 exporterMap 中获取 exporter，再从 exporter 中获取 invoker)。
                Invoker<?> invoker = getInvoker(channel, inv);
                // ...
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                // 执行服务方法，执行结果是 reply 方法的返回结果。
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: " + message == null ? null : (message.getClass().getName() + ": " + message) + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }
    	// ...
}
```



## ExchangeHandlerAdapter#reply 方法

接收 client 端传来的 Invocation 参数，交给本地执行的 invoker 处理，invoker 的返回结果是 reply 方法的返回结果。

1. 获取到 Invocation 参数后，调用 getInvoker 方法，先从 exporterMap 中获取 exporter，再从 exporter 中获取 invoker。
2. 获取到 invoker 后，调用 Invoker#invoke 方法，执行服务方法，执行结果是 reply 方法的返回结果。



## Exporter

com.alibaba.dubbo.rpc.Exporter

负责维护 Invoker 的生命周期。

```java
/**
 * Exporter. (API/SPI, Prototype, ThreadSafe)
 * 提供了获取 Invoker 的能力
 *
 * @see com.alibaba.dubbo.rpc.Protocol#export(Invoker)
 * @see com.alibaba.dubbo.rpc.ExporterListener
 * @see com.alibaba.dubbo.rpc.protocol.AbstractExporter
 */
public interface Exporter<T> {
    /**
     * get invoker.
     *
     * @return invoker
     */
    Invoker<T> getInvoker();

    /**
     * unexport.
     * <p>
     * <code>
     * getInvoker().destroy();
     * </code>
     */
    void unexport();
}
```
