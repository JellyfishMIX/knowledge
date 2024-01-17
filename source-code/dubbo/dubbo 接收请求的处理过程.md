# dubbo 接收请求的处理过程



## 说明

1. 本文基于 netty 3.0 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## provider 接收请求 -- netty 链路

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
                    	// 处理器 handler
                        .addLast("handler", nettyClientHandler);
                // ...
                }
            }
        });
    }
```

### InternalDecoder#decode -- 编码

org.apache.dubbo.remoting.transport.netty4.NettyCodecAdapter.InternalDecoder#decode

1. dubbo 继承了 netty 的入站事件 Handler -- ByteToMessageDecoder，这是 netty 预留的解码扩展点。

```java
    private class InternalDecoder extends ByteToMessageDecoder {

        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf input, List<Object> out) throws Exception {

            ChannelBuffer message = new NettyBackedChannelBuffer(input);

            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);

            // decode object.
            do {
                int saveReaderIndex = message.readerIndex();
                // 使用 Codec 进行解码，具体是 DubboCountCodec
                Object msg = codec.decode(channel, message);
                if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                    // 如果消息长度不足，则跳出返回
                    message.readerIndex(saveReaderIndex);
                    break;
                } else {
                    //is it possible to go here ?
                    if (saveReaderIndex == message.readerIndex()) {
                        throw new IOException("Decode without read data.");
                    }
                    if (msg != null) {
                        out.add(msg);
                    }
                }
            } while (message.readable());
        }
    }
```

### DubboCountCodec#decode -- 解码

org.apache.dubbo.rpc.protocol.dubbo.DubboCountCodec#decode

1. 消息长度不足，则跳出返回。解码正常，设置结果。

```java
    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int save = buffer.readerIndex();
        MultiMessage result = MultiMessage.create();
        do {
            // ExchangeCodec
            Object obj = codec.decode(channel, buffer);
            // 消息长度不足，则跳出返回
            if (Codec2.DecodeResult.NEED_MORE_INPUT == obj) {
                buffer.readerIndex(save);
                break;
            } else {
                // 解码正常，设置结果
                result.addMessage(obj);
                logMessageLength(obj, buffer.readerIndex() - save);
                save = buffer.readerIndex();
            }
        } while (true);
        if (result.isEmpty()) {
            return Codec2.DecodeResult.NEED_MORE_INPUT;
        }
        if (result.size() == 1) {
            return result.get(0);
        }
        return result;
    }
```

### ExchangeCodec#decode -- 处理 TCP 粘包拆包问题

org.apache.dubbo.remoting.exchange.codec.ExchangeCodec#decode

1. 校验 ChannelBuffer 中可读字节数是否小于消息头和消息体总长度，如果不够则本次不读取，留到下次再读。解决 TCP 拆包问题。
2. 从 ChannelBuffer 中读取请求数据，只读取一个请求长度的数据，解决 TCP 粘包问题。

```java
    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        // check magic number.
        // 校验魔数
        if (readable > 0 && header[0] != MAGIC_HIGH
                || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
            return super.decode(channel, buffer, readable, header);
        }
        // check length.
        // 校验可读字节数是否小于头信息长度，如果可读字节数小于头信息长度，则返回数据不够，需要更多数据。解决 TCP 拆包问题。
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // get data length.
        // 获取消息体长度
        int len = Bytes.bytes2int(header, 12);

        // When receiving response, how to exceed the length, then directly construct a response to the client.
        // see more detail from https://github.com/apache/dubbo/issues/7021.
        // 如果可读字节数小于消息头和消息体总长度。则返回数据不够，需要更多数据。解决 TCP 拆包问题。
        Object obj = finishRespWhenOverPayload(channel, len, header);
        if (null != obj) {
            return obj;
        }

        int tt = len + HEADER_LENGTH;
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // limit input stream.
        // 从 ChannelBuffer 中读取请求数据，只读取一个请求长度的数据，解决 TCP 粘包问题
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

        try {
            // 解码消息体
            return decodeBody(channel, is, header);
        } finally {
            if (is.available() > 0) {
                try {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Skip input stream " + is.available());
                    }
                    StreamUtils.skipUnusedStream(is);
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
    }
```

### DubboCountCodec#decode -- 解码

org.apache.dubbo.remoting.exchange.codec.ExchangeCodec#decodeBody

1. 解码请求数据，将请求数据转换为 RpcInvocation。
2. 事件类型和心跳类型的请求另行处理解码。

```java
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        // get request id.
        // 获取请求编号
        long id = Bytes.bytes2long(header, 4);
        if ((flag & FLAG_REQUEST) == 0) {
            // decode response.
            // 解码 Response
            // ...
        } else {
            // decode request.
            // 解码 Request
            Request req = new Request(id);
            // 版本号
            req.setVersion(Version.getProtocolVersion());
            // 双向通讯标记
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
            // 事件标记
            if ((flag & FLAG_EVENT) != 0) {
                req.setEvent(true);
            }
            try {
                Object data;
                // 事件请求的处理
                if (req.isEvent()) {
                    byte[] eventPayload = CodecSupport.getPayload(is);
                    if (CodecSupport.isHeartBeat(eventPayload, proto)) {
                        // heart beat response data is always null;
                        // 心跳请求所需的 EventData 为 null
                        data = null;
                    } else {
                        // 解码 EventData
                        data = decodeEventData(channel, CodecSupport.deserialize(channel.getUrl(), new ByteArrayInputStream(eventPayload), proto), eventPayload);
                    }
                } else {
                    // 解码请求数据，将请求数据转换为 RpcInvocation
                    data = decodeRequestData(channel, CodecSupport.deserialize(channel.getUrl(), is, proto));
                }
                // 设置 request 数据
                req.setData(data);
            } catch (Throwable t) {
                // bad request
                req.setBroken(true);
                req.setData(t);
            }
            return req;
        }
    }
```

### NettyServerHandler -- 事件处理 handler

NettyClient#initBootstrap 中设置了事件处理 handler

```java
    protected void initBootstrap(NettyClientHandler nettyClientHandler) {
        // ...
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
                    	// 事件处理 handler
                        .addLast("handler", nettyClientHandler);
                // ...
                }
            }
        });
    }
```

### NettyServerHandler#channelRead -- 获取并处理 channel

org.apache.dubbo.remoting.transport.netty4.NettyServerHandler#channelRead

1. 获取与客户端的连接通道 NettyChannel, 处理 channel

```java
	@Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 获取与客户端的连接通道 NettyChannel
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        // 处理 channel
        handler.received(channel, msg);
    }
```

### AllChannelHandler#received -- netty 链路转 dubbo 链路，请求进入业务线程池异步执行

org.apache.dubbo.remoting.transport.dispatcher.all.AllChannelHandler#received

1. dubbo 事件线程池，异步执行。从这里开始走出了 netty 链路，进入 dubbo 链路。
2. 如果是 Request 请求，且触发了线程池拒绝策略，则将线程池满的异常返回给客户端。

```java
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService executor = getPreferredExecutorService(message);
        try {
            // dubbo 事件线程池，异步执行。从这里开始走出了 netty 链路，进入 dubbo 链路
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            // 如果是 Request 请求，且触发了线程池拒绝策略，则将线程池满的异常返回给客户端
            if(message instanceof Request && t instanceof RejectedExecutionException){
                sendFeedback(channel, (Request) message, t);
                return;
            }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
```



## provider 接收请求 -- dubbo 链路

### ChannelEventRunnable -- dubbo 事件线程

org.apache.dubbo.remoting.transport.dispatcher.ChannelEventRunnable#run

1. dubbo 事件线程池中执行的任务是 ChannelEventRunnable, 不同 Channel 状态有不同的处理逻辑。
2. 当 Channel 状态是接收时，调用 DecodeHandler#received
3. dubbo 事件线程执行结束后，清除 dubbo 内部使用的 ThreadLocal

```java
@Override
    public void run() {
        InternalThreadLocal.removeAll();
        if (state == ChannelState.RECEIVED) {
            try {
                // 状态是接收, 调用 DecodeHandler#received
                handler.received(channel, message);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is " + message, e);
            }
        } else {
            // ...
        }
        // dubbo 事件线程执行结束后，清除 dubbo 内部使用的 ThreadLocal
        InternalThreadLocal.removeAll();
    }
```

### DecodeHandler#received -- 处理事件

org.apache.dubbo.remoting.transport.DecodeHandler#received

1. 逻辑很简单，调用 HeaderExchangeHandler#received

```java
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        // ...

        // 调用 HeaderExchangeHandler#received
        handler.received(channel, message);
    }
```

### HeaderExchangeHandler#received -- 处理事件

org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#received

1. 针对 Request 类型的不同，event，双向通讯，单向通讯，有不同的处理逻辑。

```java
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        if (message instanceof Request) {
            // handle request.
            Request request = (Request) message;
            if (request.isEvent()) {
                // 处理 event
                handlerEvent(channel, request);
            } else {
                // 处理双向通讯请求
                if (request.isTwoWay()) {
                    handleRequest(exchangeChannel, request);
                } else {
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {
            // ...
        } else {
            handler.received(exchangeChannel, message);
        }
    }
```

### HeaderExchangeHandler#handleRequest -- 处理双向通讯 Request

org.apache.dubbo.remoting.exchange.support.header.HeaderExchangeHandler#handleRequest

1. 构建返回结果 Response(通过 requestId 与 Request 做关联)
2. 调用 ExchangeHandlerAdapter#reply, 得到具体服务的返回结果
3. future 有结果时，触发以下逻辑:
   1. 没有异常，将 status 置为 OK，并将服务返回结果填入 Response
   2. 有异常，则将结果状态置为 SERVICE_ERROR，并赋值异常信息
   3. 通过 Channel 将结果写回给 consumer(即返回结果的发送操作由 dubbo 事件线程执行)

```java
    void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
        // 构建返回结果 Response(通过 requestId 与 Request 做关联)
        Response res = new Response(req.getId(), req.getVersion());
        if (req.isBroken()) {
            // ...
        }
        // find handler by message class.
        Object msg = req.getData();
        try {
            // 调用 ExchangeHandlerAdapter#reply, 得到具体服务的返回结果
            CompletionStage<Object> future = handler.reply(channel, msg);
            // future 有结果时，触发以下逻辑
            future.whenComplete((appResult, t) -> {
                try {
                    if (t == null) {
                        // 没有异常，将 status 置为 OK，并将服务返回结果填入 Response
                        res.setStatus(Response.OK);
                        res.setResult(appResult);
                    } else {
                        // 有异常，则将结果状态置为 SERVICE_ERROR，并赋值异常信息
                        res.setStatus(Response.SERVICE_ERROR);
                        res.setErrorMessage(StringUtils.toString(t));
                    }
                    // 通过 Channel 将结果写回给 consumer(即返回结果的发送操作由 dubbo 事件线程执行)
                    channel.send(res);
                } catch (RemotingException e) {
                    logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
                }
            });
        } catch (Throwable e) {
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
            channel.send(res);
        }
    }
```

### ExchangeHandlerAdapter#reply -- 响应

org.apache.dubbo.remoting.exchange.support.ExchangeHandlerAdapter#reply

1. 检查请求是否为 dubbo 的 Invocation 格式
2. 获取本地调用的 Invoker, 此处获取到的 Invoker 为 FilterChainBuilder.CallbackRegistrationInvoker
3. CallbackRegistrationInvoker#invoke 会执行 dubbo 内部和用户自定义的 Filter, 执行完 Filter 后，最终调用到 AbstractProxyInvoker#doInvoke 方法

```java
        @Override
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
            // 检查请求是否为 dubbo 的 Invocation 格式
            if (!(message instanceof Invocation)) {
                throw new RemotingException(channel, "Unsupported request: "
                    + (message == null ? null : (message.getClass().getName() + ": " + message))
                    + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
            }

            Invocation inv = (Invocation) message;
            // 获取本地调用的 Invoker, 此处获取到的 Invoker 为 FilterChainBuilder.CallbackRegistrationInvoker
            Invoker<?> invoker = getInvoker(channel, inv);
            inv.setServiceModel(invoker.getUrl().getServiceModel());
            // switch TCCL
            if (invoker.getUrl().getServiceModel() != null) {
                Thread.currentThread().setContextClassLoader(invoker.getUrl().getServiceModel().getClassLoader());
            }
            // ...
            RpcContext.getServiceContext().setRemoteAddress(channel.getRemoteAddress());
            // CallbackRegistrationInvoker#invoke 会执行 dubbo 内部和用户自定义的 Filter
            // 执行完 Filter 后，最终调用到 AbstractProxyInvoker#doInvoke 方法
            Result result = invoker.invoke(inv);
            return result.thenApply(Function.identity());
        }
```

### DubboProtocol#getInvoker -- 获取提供服务的 Invoker

org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#getInvoker

1. 组装 serviceKey,  获取 serviceKey 对应的 exporter, 调用 exporter 获取 Invoker

```java
    Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        boolean isCallBackServiceInvoke;
        boolean isStubServiceInvoke;
        int port = channel.getLocalAddress().getPort();
        String path = (String) inv.getObjectAttachmentWithoutConvert(PATH_KEY);
        
        // ...
        
        // 组装 serviceKey
        String serviceKey = serviceKey(
            port,
            path,
            (String) inv.getObjectAttachmentWithoutConvert(VERSION_KEY),
            (String) inv.getObjectAttachmentWithoutConvert(GROUP_KEY)
        );
        // 获取 serviceKey 对应的 exporter
        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

        if (exporter == null) {
            throw new RemotingException(channel, "Not found exported service: " + serviceKey + " in " + exporterMap.keySet() + ", may be version or group mismatch " +
                ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + getInvocationWithoutData(inv));
        }

        // 调用 exporter 获取 Invoker
        return exporter.getInvoker();
    }
```

### JavassistProxyFactory#getInvoker -- 获取 dubbo 生成的具体服务代理类

org.apache.dubbo.rpc.proxy.javassist.JavassistProxyFactory#getInvoker

1. 此处的 wrapper 是 dubbo 生成的具体服务代理类

```java
    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        try {
            // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
            final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
            return new AbstractProxyInvoker<T>(proxy, type, url) {
                @Override
                protected Object doInvoke(T proxy, String methodName,
                                          Class<?>[] parameterTypes,
                                          Object[] arguments) throws Throwable {
                    // 此处的 wrapper 是 dubbo 生成的具体服务代理类
                    return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
                }
            };
        } catch (Throwable fromJavassist) {
            // ...
        }
    }
```

### 具体服务的代理类

1. 服务代理类是 dubbo 运行时生成的，根据方法名和方法参数，判定执行的业务方法。

```java
public class Wrapper$1
extends Wrapper
implements ClassGenerator.DC {

    // 省略其他代码...

    public Object invokeMethod(Object object, String string, Class[] classArray, Object[] objectArray) throws InvocationTargetException {
        DemoServiceImpl demoServiceImpl;
        try {
            demoServiceImpl = (DemoServiceImpl)object;
        } catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        try {
            if ("sayHello".equals(string) && classArray.length == 1) {
                return demoServiceImpl.sayHello((String)objectArray[0]);
            }
            if ("sayHello2".equals(string) && classArray.length == 1) {
                return demoServiceImpl.sayHello2((String)objectArray[0]);
            }
        } catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
        throw new NoSuchMethodException(new StringBuffer().append("Not found method \"").append(string).append("\" in class com.yuqiao.deeplearningdubbo.analysis.base.DemoServiceImpl.").toString());
    }
}
```

### AbstractProxyInvoker#invoke -- 调用具体服务的代理类获得返回结果

org.apache.dubbo.rpc.proxy.AbstractProxyInvoker#invoke

1. 执行子类实现的 doInvoke 方法，例如 JavassistProxyFactory#getInvoker 方法中实现了 AbstractProxyInvoker#doInvoke 方法，调用了具体服务的代理类 invokeMethod 方法，获得返回值。
2. 构建 Future 填充返回值，返回 Future。
3. 后续在 HeaderExchangeHandler#handleRequest 中拿到 Future 后，future 有结果时，触发返回值写入 Channel 的写操作，将返回结果返回给 consumer。
4. 这里的 Future 是个假异步，先同步调用具体服务的代理类获得返回值，再构建 Future，Future 构建出来后已经有返回值了。

```java
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            // ...

            // 执行子类实现的 doInvoke 方法
            Object value = doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());

            // ...

            CompletableFuture<Object> future = wrapWithFuture(value, invocation);
            CompletableFuture<AppResponse> appResponseFuture = future.handle((obj, t) -> {
                AppResponse result = new AppResponse(invocation);
                if (t != null) {
                    if (t instanceof CompletionException) {
                        result.setException(t.getCause());
                    } else {
                        result.setException(t);
                    }
                } else {
                    result.setValue(obj);
                }
                return result;
            });
            return new AsyncRpcResult(appResponseFuture, invocation);
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

