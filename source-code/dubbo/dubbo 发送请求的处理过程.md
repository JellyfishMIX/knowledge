# dubbo 发送请求的处理过程



## 说明

1. 本文基于 netty 3.0 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## dubbo 四种调用方式

### oneway --- 单向调用

oneway 指的是客户端发送消息后，不需要接受响应。适合应用在 client 不关注 server 端响应的场景。注意，返回值定义为 void 的并不是 oneway 的调用方式，void 表示的无需返回值，对 dubbo 框架而言，还是需要构建返回数据。

观察 oneway 调用方式的图，可以看出: 从 client 到 server，只有 req，没有 resp，客户端不需要阻塞等待。

### sync --- 同步调用

同步调用是最常用的通信方式，也是 dubbo 默认的通信模式。client 发送 req 后阻塞等待 resp 的返回。

### future -- 异步调用

client 调用 invoker 时返回一个 future, dubbo 收到 server 的返回结果后，会调用 future.complete 方法填充结果。client 在合适的时机使用 future.get 方法获取返回结果。

### callback -- 回调

callback 是 client 调用 server, 发送 seq 后 server 无需在本次请求返回 resp, 而是 server 在可以写返回结果时，通过反向调用 client 接口的方式，把返回结果作为请求参数传递给 client。这样 client 的主线程无需阻塞。

#### 回调如何保证调用到之前调用自己的同一台机器

使用之前建立的 SocketChannel，继续用就可以天然地调用到之前调用自己的同一台机器。

![img](https://image-hosting.jellyfishmix.com/20230925155302.png)



## consumer 发送请求 -- Invoker 调用链路

在 consumer 视角看来，dubbo 服务是接口，接口的实现是 dubbo 生成的代理对象，consumer 调用服务接口使用的是代理对象。分析 consumer 发送请求的机制，要从 dubbo 生成的代理类开始。

以 DemoService 为例，dubbo 生成的代理类。

1. Dubbo 生成的 Proxy0 包含了 DemoService 接口所有方法的代理方法。因此可以通过 proxy.methodName(args...) 的方式调用。
2. 其调用的核心逻辑是，当调用 DemoService#sayHello 方法时，实际调用的是 InvokerInvocationHandler#invoke，并将接口,方法和参数作为 InvokerInvocationHandler.invoke() 的参数传入进去。

```java
/**
 * DemoService
 */
public interface DemoService {
    public String sayHello(String name);
    public String sayHello2(String name);
}

/**
 * dubbo 根据 DemoService 接口，生成的 Proxy0
 */
public class Proxy0 implements ClassGenerator.DC,DemoService,EchoService {
    /**
     * DemoService 接口的所有方法
     */
    public static Method[] methods;
    /**
     * 实例化时，注入的 InvokerInvocationHandler
     */
    private InvocationHandler handler;

    /**
     * 代理 sayHello
     */
    public String sayHello(String string) {
        Object[] objectArray = new Object[]{string};
        // 以当前接口，调用的方法，方法的参数，作为参数，调用 InvokerInvocationHandler#invoke
        Object object = this.handler.invoke(this, methods[0], objectArray);
        return (String)object;
    }

    public String sayHello2(String string) {
        Object[] objectArray = new Object[]{string};
        // 以当前接口，调用的方法，方法的参数，作为参数，调用 InvokerInvocationHandler#invoke
        Object object = this.handler.invoke(this, methods[1], objectArray);
        return (String)object;
    }

    // 省略部分代码...

    /**
     * 注入 InvocationHandler
     */
    public proxy0(InvocationHandler invocationHandler) {
        this.handler = invocationHandler;
    }
}
```

### 服务调用的大致链路

consumerProxy -> InvocationHandler -> invoker -> ExchangeClient -> ExchangeChannel -> NettyChannel -> NioSocketChannel -> provider

### InvokerInvocationHandler

1. InvokerInvocationHandler 编写了调用 Invoker 的逻辑。
   1. 创建 RpcInvocation。RpcInvocation 是 dubbo rpc 调用信息的载体，里面包含了调用接口，参数，调用模式等内容。
   2. 调用 Invoker#invoke，发起远程调用。
   3. 调用 Result#recreate，获取远程返回的结果。
2. 此处的 Invoker 为 MockClusterInvoker，通过 MockClusterWrapper#join 创建。MockClusterWrapper 是通过 ExtensionLoader 创建 Cluster 的 extension 时创建的包装类。

```java
public class InvokerInvocationHandler implements InvocationHandler {
    private static final Logger logger = LoggerFactory.getLogger(InvokerInvocationHandler.class);
    /**
     * 此处的 Invoker 为 MockClusterInvoker，通过 MockClusterWrapper#join 创建
     * MockClusterWrapper 是通过 ExtensionLoader 创建 Cluster 的 extension 时创建的包装类。
     */
    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 调用的方法名
        String methodName = method.getName();
        proxy.notify();
        // 调用的方法参数
        Class<?>[] parameterTypes = method.getParameterTypes();
        // 直接调用 Object 定义的方法
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        // 直接调用 invoker 重写的 toString, hashCode 和 equals
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        // 通过 Invoker 发起远程调用
        // 1. 创建 RpcInvocation。RpcInvocation 是 dubbo rpc 调用信息的载体，里面包含了调用接口，参数，调用模式等内容。
        // 2. 调用 Invoker#invoke，发起远程调用。
        // 3. 调用 Result#recreate，获取远程返回的结果。
        // 这里默认的 invoker 是通过 Cluster 包装类 MockClusterWrapper 创建的 MockClusterInvoker，
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
}
```

为什么在 proxy 和 Invoker 之间有一层 InvokerInvocationHandler 呢？

InvokerInvocationHandler 实现了接口 java.lang.reflect.InvocationHandler, 拥有作为 jdk 动态代理机制 handler 的能力，可以自定义生成的代理类方法被调用时的逻辑。对于 dubbo consumer 来说，在 InvokerInvocationHandler 中编写的是调用 Invoker 的逻辑。

```java
/**
 * 实现了接口 java.lang.reflect.InvocationHandler
 */
public class InvokerInvocationHandler implements InvocationHandler {}
```

InvokerInvocationHandler 被用于创建代理类时作为 handler。

![image-20230927113145092](https://image-hosting.jellyfishmix.com/20230927113145.png)

### MockClusterInvoker#invoke

org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker#invoke

1. 封装了 mock 的逻辑，默认为非 mock，调用远程服务。默认为 FailoverClusterInvoker，invoke 在其父类 AbstractClusterInvoker 中实现的。

```java
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        // mock 开关，默认为 false
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), Constants.MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            //no mock
            // 非 mock，调用远程服务。默认为 FailoverClusterInvoker
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            if (logger.isWarnEnabled()) {
                logger.info("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
            }
            //force:direct mock
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            try {
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                } else {
                    if (logger.isWarnEnabled()) {
                        logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                    }
                    result = doMockInvoke(invocation, e);
                }
            }
        }
        return result;
    }
```

### AbstractClusterInvoker

org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#invoke

invoke 方法在其父类 AbstractClusterInvoker 中实现的。

1. 检查是否已销毁。
3. 获取可用的 Invokers, 来自于 Directory#list 方法。所有的 invokers 都保存在 RegistryDirectory#methodInvokerMap 中。
4. 初始化负载均衡。
5. 调用子类的模板方法 doInvoke()。

```java
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        // 检查是否已销毁
        checkWhetherDestroyed();
        
        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Router route.");
        /*
         * 获取可用的 Invokers, 来自于 Directory#list 方法
         * 所有的 invokers 都保存在 RouterChain 中
         */
        List<Invoker<T>> invokers = list(invocation);
        InvocationProfilerUtils.releaseDetailProfiler(invocation);

        // 初始化负载均衡
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        // 为异步调用添加 InvocationId
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Cluster " + this.getClass().getName() + " invoke.");
        try {
            // 调用子类的模板方法 doInvoke()
            return doInvoke(invocation, invokers, loadbalance);
        } finally {
            InvocationProfilerUtils.releaseDetailProfiler(invocation);
        }
    }
```

### AbstractClusterInvoker#list 方法 -- 获取可用的 Invokers

org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#list

```java
protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
    // 通过服务目录 Directory，获取可用的 Invoker
    return directory.list(invocation);
}
```

### AbstractDirectory#list 方法 -- 获取可用的 Invokers 集合

```java
public List<Invoker<T>> list(Invocation invocation) throws RpcException {
    if (destroyed) {
        throw new RpcException("Directory already destroyed .url: " + getUrl());
    }
    // 调用子类的模板方法 doList，获取可用的 Invoker
    return doList(invocation);
}
```

### DynamicDirectory#doList 方法 -- 获取可用的 Invokers 集合(可以做路由机制)

org.apache.dubbo.registry.integration.DynamicDirectory#doList

1. 从 RouterChain 中获取invokers，内含过滤逻辑。这里可以做路由机制，比如根据自定义 id 参数，仅选择特定 id 环境的 invokers(qunar 的软路由理论依据~)。
2. 此处可以看到日常调试过程中常见的问题，No provider available from registry。

```java
    @Override
    public List<Invoker<T>> doList(BitList<Invoker<T>> invokers, Invocation invocation) {
        // 日常调试过程中常见的问题，No provider available from registry
        if (forbidden && shouldFailFast) {
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "No provider available from registry " +
                getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +
                NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() +
                ", please check status of providers(disabled, not registered or in blacklist).");
        }

        if (multiGroup) {
            return this.getInvokers();
        }

        try {
            // Get invokers from cache, only runtime routers will be executed.
            // 从 RouterChain 中获取 invokers，内含过滤逻辑。这里可以做路由机制，比如根据自定义 id 参数，仅选择特定 id 环境的 invokers(qunar 的软路由理论依据~)。
            List<Invoker<T>> result = routerChain.route(getConsumerUrl(), invokers, invocation);
            return result == null ? BitList.emptyList() : result;
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            return BitList.emptyList();
        }
    }
```

### FailoverClusterInvoker#doInvoke 方法 -- 执行调用

org.apache.dubbo.rpc.cluster.support.FailoverClusterInvoker#doInvoke

1. 调用失败后，重试指定次数。
2. 重试前，考虑到 invokers 可能有变更，重新获取可用的 invokers 集合，并再次校验。
3. 根据负载均衡策略，选出要调用的 invoker, 用于执行远程调用。
4. 调用 Invoker#invoke 方法，执行远程调用。

```java
    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyinvokers = invokers;
        checkInvokers(copyinvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        // 获取重试次数
        int len = getUrl().getMethodParameter(methodName, Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        // 调用失败后，重试指定次数
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            // 重试前，考虑到 invokers 可能有变更，重新获取可用的 invokers，并再次校验
            if (i > 0) {
                checkWhetherDestroyed();
                copyinvokers = list(invocation);
                // check again
                checkInvokers(copyinvokers, invocation);
            }
            // 根据负载均衡策略, 选出胜出的 Invoker，用于执行远程调用
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 调用 Invoker#invoke，发起远程调用。最终调用 AsyncToSyncInvoker#invoke
                Result result = invoker.invoke(invocation);
                // ...
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        
        // ...
    }
```

### 根据负载均衡策略，从 invoker 列表中选择要调用的 invoker -- AbstractClusterInvoker#select

org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#select

1. 重点是 doSelect 方法，根据使用的 LoadBalance 策略，从 invoker 列表中选择要调用的 invoker。

```java
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation,
                                List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {

        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        String methodName = invocation == null ? StringUtils.EMPTY_STRING : invocation.getMethodName();

        boolean sticky = invokers.get(0).getUrl()
            .getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);

        //ignore overloaded method
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        //ignore concurrency problem
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            if (availableCheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }

        Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

        if (sticky) {
            stickyInvoker = invoker;
        }

        return invoker;
    }
```

#### 举例，默认的负载均衡策略，随机选取 Invokers -- RandomLoadBalance

org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance#doSelect

分两种情况 invoker 权重一样/不一样:

1. 权重一样: 随机取一个 invoker 返回。
2. 权重不一样: 
   1. 产出一个区间[0, totalWeight)内的随机数，根据产出的随机数，获取 invoker。
   2. 举例，有 A,B 两个服务，权重分别是 2,3
   3. 权重值之和为 5，随机数区间为[0,1,2,3,4]，A 服务被选中的概率为 2/5，取值范围[0,1]，B 服务被选中的概率为 3/5，取值范围[2,3,4]
   4. 如果随机数为 4，则命中 B 服务。如果随机数为 1，则命中 A 服务。

```java
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // invokers 总数
        int length = invokers.size(); // Number of invokers
        // 所有 invoker 的权重值之和
        int totalWeight = 0; // The sum of weights
        // 每个 invoker 的权重是否一样
        boolean sameWeight = true; // Every invoker has the same weight?
        for (int i = 0; i < length; i++) {
            // 第一个 invoker 的权重值
            int weight = getWeight(invokers.get(i), invocation);
            totalWeight += weight; // Sum
            // 校验权重实际是否一样
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }

        // invoker 权重值不同的计算方法
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            // 产出一个区间[0, totalWeight)内的随机数
            int offset = random.nextInt(totalWeight);
            // Return a invoker based on the random value.
            /*
             * 根据产出的随机数，获取 invoker
             * 举例，有 A,B 两个服务，权重分别是 2,3
             * 权重值之和为 5，随机数区间为[0,1,2,3,4]，A 服务被选中的概率为 2/5，取值范围[0,1]，B 服务被选中的概率为 3/5，取值范围[2,3,4]
             * 如果随机数为 4，则命中 B 服务。如果随机数为 1，则命中 A 服务。
             */
            for (int i = 0; i < length; i++) {
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        // 权重一样，随机取一个 invoker 返回
        return invokers.get(random.nextInt(length));
    }
```

### AbstractInvoker#invoke 方法 -- 选中 invoker 后的调用逻辑

```java
    @Override
    public Result invoke(Invocation inv) throws RpcException {
        // ...

        RpcInvocation invocation = (RpcInvocation) inv;

        // prepare rpc invocation
        prepareInvocation(invocation);

        // do invoke rpc invocation and return async result
        AsyncRpcResult asyncResult = doInvokeAndReturn(invocation);

        // wait rpc result if sync
        waitForResultIfSync(asyncResult, invocation);

        return asyncResult;
    }
```

### AbstractInvoker#waitForResultIfSync 方法 -- 异步转同步

检查 InvokeMode，如果是同步模式，就调用 CompletableFuture#get(timeout) 有参方法阻塞等待。

tips: dubbo 注释里不使用 CompletableFuture#get() 无参方法做死等的原因，可以看这篇文章 [CompletableFuture 早期版本的性能问题](https://juejin.cn/post/7016919183510208543)

```java
    private void waitForResultIfSync(AsyncRpcResult asyncResult, RpcInvocation invocation) {
        if (InvokeMode.SYNC != invocation.getInvokeMode()) {
            return;
        }
        try {
            /*
             * NOTICE!
             * must call {@link java.util.concurrent.CompletableFuture#get(long, TimeUnit)} because
             * {@link java.util.concurrent.CompletableFuture#get()} was proved to have serious performance drop.
             */
            Object timeout = invocation.getObjectAttachmentWithoutConvert(TIMEOUT_KEY);
            if (timeout instanceof Integer) {
                asyncResult.get((Integer) timeout, TimeUnit.MILLISECONDS);
            } else {
                asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
            }
        } catch (InterruptedException e) {
            // ...
        }
    }
```

AbstractInvoker 的子类，这几种子类均具有异步转同步的能力。

![image-20230925190728401](https://image-hosting.jellyfishmix.com/20230925190728.png)

### AbstractInvoker#doInvokeAndReturn -- 执行子类实现的调用

org.apache.dubbo.rpc.protocol.AbstractInvoker#doInvokeAndReturn

```java
    private AsyncRpcResult doInvokeAndReturn(RpcInvocation invocation) {
        AsyncRpcResult asyncResult;
        try {
            // 执行子类实现的调用
            asyncResult = (AsyncRpcResult) doInvoke(invocation);
        } catch (InvocationTargetException e) {
            // ...
        }

        if (setFutureWhenSync || invocation.getInvokeMode() != InvokeMode.SYNC) {
            // set server context
            RpcContext.getServiceContext().setFuture(new FutureAdapter<>(asyncResult.getResponseFuture()));
        }

        return asyncResult;
    }
```

### DubboInvoker#doInvoke -- Invoker 调用细节

org.apache.dubbo.rpc.protocol.dubbo.DubboInvoker#doInvoke

1. 使用了抽象的网络 IO 组件 ExchangeClient，常见的组合实现是 Netty。
2. 单向调用执行的是 ExchangeClient#send，双向调用执行的是 ExchangeClient#request。

```java
    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);

        // 抽象了网络 IO 组件
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            // 判断是否单程调用
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = calculateTimeout(invocation, methodName);
            invocation.setAttachment(TIMEOUT_KEY, timeout);
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                // 单向调用执行的是 ExchangeClient#send
                currentClient.send(inv, isSent);
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                ExecutorService executor = getCallbackExecutor(getUrl(), inv);
                // 双向调用执行的是 ExchangeClient#request
                CompletableFuture<AppResponse> appResponseFuture =
                        currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(appResponseFuture);
                AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
                result.setExecutor(executor);
                return result;
            }
        } catch (TimeoutException e) {
            // ...
        }
    }
```

### HeaderExchangeChannel#send 方法 -- 发送请求无返回值

org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#send

1. 调用网络 IO 组件 Channel 发送，默认使用 netty 版实现 NettyClient。

```java
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send message " + message + ", cause: The channel " + this + " is closed!");
        }
        if (message instanceof Request
                || message instanceof Response
                || message instanceof String) {
            channel.send(message, sent);
        } else {
            Request request = new Request();
            request.setVersion(Version.getProtocolVersion());
            request.setTwoWay(false);
            request.setData(message);
            // 调用网络 IO 组件 Channel#send 发送，默认使用 netty 版实现 NettyClient
            channel.send(request, sent);
        }
    }
```

NettyClient 是实现

![image-20230927143740740](https://image-hosting.jellyfishmix.com/20230927143741.png)



### HeaderExchangeClient#request -- 发送请求有返回值

org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeChannel#request

```java
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setData(request);
        // 构建 DefaultFuture, 继承自 CompletableFuture
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
        try {
            // 调用网络 IO 组件 Channel#send 发送，默认使用 netty 版实现 NettyClient
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```

### DefaultFuture#newFuture 方法 -- 构建 DefaultFuture(继承自 CompletableFuture)

org.apache.dubbo.remoting.exchange.support.DefaultFuture#newFuture

1. 构造 DefaultFuture, 构造时关联了 requestId 和 DefaultFuture。
2. requestId 和 DefaultFuture 关联，IO 组件收到 server 端返回结果后，就能关联到 future 了。

```java
    /**
     * init a DefaultFuture
     * 1.init a DefaultFuture
     * 2.timeout check
     *
     * @param channel channel
     * @param request the request
     * @param timeout timeout
     * @return a new DefaultFuture
     */
    public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
        // 构造 DefaultFuture, 构造时关联了 requestId 和 DefaultFuture
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        future.setExecutor(executor);
        // ThreadlessExecutor needs to hold the waiting future in case of circuit return.
        if (executor instanceof ThreadlessExecutor) {
            ((ThreadlessExecutor) executor).setWaitingFuture(future);
        }
        // timeout check
        timeoutCheck(future);
        return future;
    }

    private DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
        // put into waiting map.
        // requestId 和 DefaultFuture 关联，IO 组件收到 server 端返回结果后，就能关联到 future 了
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }

	private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();
```

### 网络 IO 使用 NettyChannel

org.apache.dubbo.remoting.transport.netty.NettyChannel#send

1. 以默认的实现 NettyChannel 举例，IO 操作调用了 netty 的组件 Channel，是对 java NIO 中的 SocketChannel 的封装。

```java
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        super.send(message, sent);

        boolean success = true;
        int timeout = 0;
        try {
            // netty 的 Channel 组件，是对 java NIO 中的 SocketChannel 的封装
            ChannelFuture future = channel.write(message);
            if (sent) {
                timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
                success = future.await(timeout);
            }
            Throwable cause = future.getCause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            // ...
        }
    }
```



## consumer 发送请求 -- netty 链路

org.apache.dubbo.remoting.transport.netty4.NettyClient#initBootstrap

1. 配置 netty 的启动器 bootstrap。
2. 给 Channel 对应的 Pipeline 设置解码器和编码器。

```java
    protected void initBootstrap(NettyClientHandler nettyClientHandler) {
        // bootstrap 是 netty 的启动器，配置 netty
        bootstrap.group(EVENT_LOOP_GROUP.get())
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(socketChannelClass());

        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, Math.max(DEFAULT_CONNECT_TIMEOUT, getConnectTimeout()));
        bootstrap.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    ch.pipeline().addLast("negotiation", new SslClientTlsHandler(getUrl()));
                }
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        // 设置解码器
                        .addLast("decoder", adapter.getDecoder())
                        // 设置编码器
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                        .addLast("handler", nettyClientHandler);
                // ...
                }
            }
        });
    }
```

### InternalEncoder#encode -- 编码

org.apache.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalEncoder#encode

1. dubbo 继承了 netty 的出站事件 Handler -- MessageToByteEncoder，这是 netty 预留的编码扩展点。

```java
    private class InternalEncoder extends MessageToByteEncoder {

        @Override
        protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
            ChannelBuffer buffer = new NettyBackedChannelBuffer(out);
            Channel ch = ctx.channel();
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            // 使用 Codec 进行编码
            codec.encode(channel, buffer, msg);
        }
    }
```

### ExchangeCodec#encode -- 编码

org.apache.dubbo.remoting.exchange.codec.ExchangeCodec#encode

1. 编码数据分为请求数据和响应数据两类。

```java
    @Override
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        // 编码请求数据
        if (msg instanceof Request) {
            encodeRequest(channel, buffer, (Request) msg);
        } else if (msg instanceof Response) {
            // 编码响应数据
            encodeResponse(channel, buffer, (Response) msg);
        } else {
            super.encode(channel, buffer, msg);
        }
    }
```

### ExchangeCodec#encodeRequest -- 编码请求数据

org.apache.dubbo.remoting.exchange.codec.ExchangeCodec#encodeRequest

1. 准备 header 信息。
2. 将 buffer 的写指针位置指向原写位置 + 头信息长度，即预留出 header 的空间，先写入请求体。这样写入请求体后可以获得 header 需要的请求体长度信息。
3. 序列化，具体编码工作。
4. 向 header 填充请求体长度，重置写指针位置为 header 起始，写入 header。
5. ChannelBuffer 写指针位置移动至请求体末尾。

```java
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        Serialization serialization = getSerialization(channel, req);
        // header.
        // 设置头信息
        byte[] header = new byte[HEADER_LENGTH];
        // set magic number.
        // 设置魔数
        Bytes.short2bytes(MAGIC, header);

        // set request and serialization flag.
        // 设置请求/响应标记
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());

        // 设置是否双向通讯
        if (req.isTwoWay()) {
            header[2] |= FLAG_TWOWAY;
        }
        // 设置事件类型
        if (req.isEvent()) {
            header[2] |= FLAG_EVENT;
        }

        // set request id.
        // 设置请求编号
        Bytes.long2bytes(req.getId(), header, 4);

        // encode request data.
        // ChannelBuffer 写指针
        int savedWriteIndex = buffer.writerIndex();
        // 将 buffer 的写指针位置指向原写位置 + 头信息长度，即预留出 header 的空间，先写入请求体。这样写入请求体后可以获得 header 需要的请求体长度信息。
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);

        if (req.isHeartbeat()) {
            // heartbeat request data is always null
            bos.write(CodecSupport.getNullBytesOf(serialization));
        } else {
            // 序列化
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            if (req.isEvent()) {
                // 对事件数据编码
                encodeEventData(channel, out, req.getData());
            } else {
                // 对请求数据编码
                encodeRequestData(channel, out, req.getData(), req.getVersion());
            }
            out.flushBuffer();
            if (out instanceof Cleanable) {
                ((Cleanable) out).cleanup();
            }
        }

        bos.flush();
        bos.close();
        // 向 header 填充请求体长度
        int len = bos.writtenBytes();
        checkPayload(channel, len);
        Bytes.int2bytes(len, header, 12);

        // 重置写指针位置为 header 起始，写入 header
        buffer.writerIndex(savedWriteIndex);
        buffer.writeBytes(header); // write header.
        // ChannelBuffer 写指针位置移动至请求体末尾
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
```

dubbo 编码过程是先写请求体，计算出请求体长度后，再写 header。

![image.png](https://image-hosting.jellyfishmix.com/20231004015228.png)



## Q&A

1. dubbo 的 AbstractInvoker#invoke 方法封装的返回结果是一个 Future，既然用到了 Future，说明走了 java NIO 机制的异步请求。对于 dubbo 的同步调用方式，java NIO 机制的 SocketChannel 有了返回结果后，如何与对应的请求(更准确地说是与对应请求的 Future)关联起来，以调用 CompletableFuture#complete 等 api 去完成 Future 呢？
   1. 答: dubbo 发送请求时拿到的 DefaultFuture(继承自 CompletableFuture)，DefaultFuture 在构造函数中调用 FUTURES.put(id, this); 把 requestId-DefaultFuture 的对应关系存入了 FUTURES 这个 map 中。
