# netty 建立连接源码分析



## 说明

1. 本文基于 netty 4.1 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## doBind 方法 -- 建立连接

io.netty.bootstrap.AbstractBootstrap#doBind

1. 初始化 ServerSocketChannel，把对应的网络事件注册到 Selector。
2. 给 ServerSocketChannel 绑定端口。

```java
    /**
     * Create a new {@link Channel} and bind it.
     */
    public ChannelFuture bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
    }

	private ChannelFuture doBind(final SocketAddress localAddress) {
        // 初始化 ServerSocketChannel，把对应的网络事件注册到 Selector
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            // 给 ServerSocketChannel 绑定端口
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // ...
        }
    }
```

### AbstractBootstrap#initAndRegister 方法 -- 初始化 ServerSocketChannel 并注册

io.netty.bootstrap.AbstractBootstrap#initAndRegister

1. 通过工厂类获取 ServerSocketChannel 并初始化。
2. 获取启动时创建的 EventLoopGroup, 然后把 channel 注册到 Selector 上，目的是在 EventLoopGroup 中循环获取 channel 上的网络事件。

```java
    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 通过工厂类获取 ServerSocketChannel
            channel = channelFactory.newChannel();
            // 初始化 ServerSocketChannel
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        // 获取启动时创建的 EventLoopGroup, 然后把 channel 注册到 Selector 上，关注连接事件。目的是在 EventLoopGroup 中循环获取 channel 上的网络事件。
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        return regFuture;
    }
```

### ServerBootstrap#init 方法 -- 添加 ServerBootstrapAcceptor 并关注读事件

io.netty.bootstrap.ServerBootstrap#init

io.netty.bootstrap.ServerBootstrap.ServerBootstrapAcceptor#channelRead

1. 给 channel 设置参数，设置相关属性。
2. 获取 channel 的 pipeline，向 pipeline 中添加 ChannelHandler。
   1. 向 pipeline 中添加一个特殊的 ChannelHandler，用于初始化 channel。这是第一个 ChannelHandler 。
   2. 向 pipeline 中添加一个内部的 ChannelHandler。这是第二个 ChannelHandler 。
   3. 加入一个名为 ServerBootstrapAcceptor 的 ChannelHandler。这是第三个 ChannelHandler 。
3. ServerBootstrapAcceptor 中会向 pipeline 添加用户设置的 ChannelHandler。

![Netty服务端连接初始化的四个拦截器.png](https://image-hosting.jellyfishmix.com/20230630153128.png)

```java
    @Override
    void init(Channel channel) {
        // 给 channel 设置参数
        setChannelOptions(channel, newOptionsArray(), logger);
        // 设置相关属性
        setAttributes(channel, newAttributesArray());

        // 获取 channel 的 pipeline
        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);

        // 向 pipeline 中添加一个特殊的 ChannelHandler, 用于初始化 channel
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) {
                final ChannelPipeline pipeline = ch.pipeline();
                // 向 pipeline 中添加一个内部的 ChannelHandler
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                // 加入一个名为 ServerBootstrapAcceptor 的 ChannelHandler
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }

        @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            // 用户设置的 ChannelHandler
            child.pipeline().addLast(childHandler);

            setChannelOptions(child, childOptions, logger);
            setAttributes(child, childAttrs);

            try {
                // register 方法把读事件注册到 Selector 上
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

### AbstractNioChannel#doRegister 方法 -- 向 Selector 注册 Channel 并关注连接事件

io.netty.channel.nio.AbstractNioChannel#doRegister

1. 把 ServerSocketChannel 以及关注的 OP_ACCEPT(ops=0) 事件注册到 Selector 上。

```java
    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                // 把 ServerSocketChannel 以及关注的 OP_ACCEPT(ops=0) 事件注册到 Selector 上
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```



## MultithreadEventLoopGroup -- EventLoop 线程池

如果进行未指定，默认线程数是 CPU 的核数*2。

对于子类实现 NioEventLoopGroup，新建一个子线程会调用 newChild 方法。

```java
    static {
        // 默认线程数为 CPU 核数的两倍
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }

    /**
     * 支持指定线程数
     * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor, Object...)
     */
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
```

### NioEventLoopGroup#newChild -- 新建一个子线程

io.netty.channel.nio.NioEventLoopGroup#newChild

新建了一个线程 NioEventLoop，子线程创建后，会在 NioEventLoop#run 方法中循环持续尝试获取网络事件。

```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        SelectorProvider selectorProvider = (SelectorProvider) args[0];
        SelectStrategyFactory selectStrategyFactory = (SelectStrategyFactory) args[1];
        RejectedExecutionHandler rejectedExecutionHandler = (RejectedExecutionHandler) args[2];
        EventLoopTaskQueueFactory taskQueueFactory = null;
        EventLoopTaskQueueFactory tailTaskQueueFactory = null;

        int argsLength = args.length;
        if (argsLength > 3) {
            taskQueueFactory = (EventLoopTaskQueueFactory) args[3];
        }
        if (argsLength > 4) {
            tailTaskQueueFactory = (EventLoopTaskQueueFactory) args[4];
        }
        // NioEventLoop 是线程组里的一个线程
        return new NioEventLoop(this, executor, selectorProvider,
                selectStrategyFactory.newSelectStrategy(),
                rejectedExecutionHandler, taskQueueFactory, tailTaskQueueFactory);
    }
```

### NioEventLoop#register0 方法 -- 注册 channel 关注的网络事件

io.netty.channel.nio.NioEventLoop#register0

```java
    private void register0(SelectableChannel ch, int interestOps, NioTask<?> task) {
        try {
            // 注册 channel 关注的网络事件
            ch.register(unwrappedSelector, interestOps, task);
        } catch (Exception e) {
            throw new EventLoopException("failed to register a channel", e);
        }
    }
```



## NioEventLoop#run 方法 -- 循环持续尝试获取网络事件

io.netty.channel.nio.NioEventLoop#run

1. 设定循环的间隔时间，调用 java NIO 尝试获取网络事件。

```java
    @Override
    protected void run() {
        int selectCnt = 0;
        // 循环持续尝试获取网络事件
        for (;;) {
            try {
                int strategy;
                try {
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;

                    case SelectStrategy.BUSY_WAIT:
                        // fall-through to SELECT since the busy-wait is not supported with NIO

                    case SelectStrategy.SELECT:
                        // 设定循环的间隔时间
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE; // nothing on the calendar
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                // 调用 java NIO 尝试获取网络事件
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            // This update is just to help block unnecessary selector wakeups
                            // so use of lazySet is ok (no race condition)
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                        // fall through
                    default:
                    }
                } catch (IOException e) {
                    // If we receive an IOException here its because the Selector is messed up. Let's rebuild
                    // the selector and retry. https://github.com/netty/netty/issues/8566
                    rebuildSelector0();
                    selectCnt = 0;
                    handleLoopException(e);
                    continue;
                }

                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        // Ensure we always run tasks.
                        ranTasks = runAllTasks();
                    }
                } else if {
                    // ...
                }
            }
            // ....
        }
    }
```

### NioEventLoop#select 方法 -- 通过 javaNIO select 获取网络事件

io.netty.channel.nio.NioEventLoop#select

1. 通过 javaNIO select 获取网络事件，未设置等待时间(即循环间隔时间)则采用无阻塞的 IO，设置了等待时间则采用阻塞的 IO。

```java
    private int select(long deadlineNanos) throws IOException {
        // 未设置等待时间(即循环间隔时间)则采用无阻塞的 IO
        if (deadlineNanos == NONE) {
            // 通过 javaNIO select 获取网络事件
            return selector.select();
        }
        // 设置了等待时间则采用阻塞的 IO，通过 javaNIO select 获取网络事件
        // Timeout will only be 0 if deadline is within 5 microsecs
        long timeoutMillis = deadlineToDelayNanos(deadlineNanos + 995000L) / 1000000L;
        return timeoutMillis <= 0 ? selector.selectNow() : selector.select(timeoutMillis);
    }
```

### NioEventLoop#processSelectedKeysPlain 方法 -- 循环处理注册在 selector 上的所有事件

io.netty.channel.nio.NioEventLoop#processSelectedKeysPlain

1. 循环处理注册在 selector 上的所有事件。
2. 事件遍历完了就退出。

```java
    private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
        if (selectedKeys.isEmpty()) {
            return;
        }

        // 循环处理注册在 selector 上的所有事件
        Iterator<SelectionKey> i = selectedKeys.iterator();
        for (;;) {
            final SelectionKey k = i.next();
            final Object a = k.attachment();
            i.remove();

            if (a instanceof AbstractNioChannel) {
                // 对事件进行处理
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }

            // 事件遍历完了就退出
            if (!i.hasNext()) {
                break;
            }

            if (needsToSelectAgain) {
                selectAgain();
                selectedKeys = selector.selectedKeys();

                // Create the iterator again to avoid ConcurrentModificationException
                if (selectedKeys.isEmpty()) {
                    break;
                } else {
                    i = selectedKeys.iterator();
                }
            }
        }
    }
```



## NioEventLoop#processSelectedKey 方法 -- 处理一个网络事件

io.netty.channel.nio.NioEventLoop#processSelectedKey(java.nio.channels.SelectionKey, io.netty.channel.nio.AbstractNioChannel)

1. 根据事件类型进行处理。
2. 对 OP_READ 或 OP_ACCEPT 事件的处理，用到的方法是 AbstractNioChannel.NioUnsafe#read，实现类是 AbstractNioMessageChannel

```java
    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                return;
            }
            if (eventLoop == this) {
                // close the channel if the key is not valid anymore
                unsafe.close(unsafe.voidPromise());
            }
            return;
        }

        try {
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
               unsafe.forceFlush();
            }

            // 对 OP_READ 或 OP_ACCEPT 事件的处理
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

### AbstractNioMessageChannel.NioMessageUnsafe#read 方法 -- 对 OP_READ 或 OP_ACCEPT 事件的处理

io.netty.channel.nio.AbstractNioMessageChannel.NioMessageUnsafe#read

```java
    private final class NioMessageUnsafe extends AbstractNioUnsafe {

        /**
         * 用来缓存连接的 list
         */
        private final List<Object> readBuf = new ArrayList<Object>();

        @Override
        public void read() {
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
            allocHandle.reset(config);

            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                    do {
                        // 建立连接并放入 readBuf 中
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        allocHandle.incMessagesRead(localRead);
                    } while (continueReading(allocHandle));
                } catch (Throwable t) {
                    exception = t;
                }

                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;
                    pipeline.fireChannelRead(readBuf.get(i));
                }
                readBuf.clear();
                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                // ...
            } finally {
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```

### NioServerSocketChannel#doReadMessages 方法 -- 建立连接并放进入参 buf 中

io.netty.channel.socket.nio.NioServerSocketChannel#doReadMessages

1. 与客户端建立连接，获得对应的 SocketChannel。
2. 把创建好的连接放进入参 buf 中。

```java
    @Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        // 与客户端建立连接，获得对应的 SocketChannel
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                // 把创建好的连接放进入参 buf 中
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            // ...
        }

        return 0;
    }

	/**
	 * io.netty.util.internal.SocketUtils#accept
	 */
	public static SocketChannel accept(final ServerSocketChannel serverSocketChannel) throws IOException {
        try {
            return AccessController.doPrivileged(new PrivilegedExceptionAction<SocketChannel>() {
                @Override
                public SocketChannel run() throws IOException {
                    // 调用 ServerSocketChannel#accept 方法建立连接
                    return serverSocketChannel.accept();
                }
            });
        } catch (PrivilegedActionException e) {
            throw (IOException) e.getCause();
        }
    }
```



## netty OP_ACCEPT,OP_READ 事件注册过程

https://blog.csdn.net/u013857458/article/details/82720569