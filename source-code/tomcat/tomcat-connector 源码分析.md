# tomcat-connector 源码分析



## 说明

1. 本文基于 tomcat 8.5.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## ProtocolHandler

1. ProtocolHandler 是一个接口，不同协议使用不同的实现，例如 http 1.1 的实现 Http11NioProtocol。



## Endpoint

1. Endpoint 负责 tomcat-connector 连接器模块下三大核心功能之一，主要负责网络通信。tomcat-connector 模块的另外两个核心功能分别是: 应用层协议解析-Processor, tomcat Request/Response 与 ServletRequest/ServletResponse 的转化-Adapter。
2. Endpoint 是个抽象类，具体实现和 ProtocolHandler 的具体实现绑定，如 Http11NioProtocol 使用的类型就是写死的 NioEndpoint 类型。



## tomcat 的 NIO 模型

1. tomcat 提供的 NIO 有两种，NioEndpoint 和 Nio2Endpoint。

2. NioEndpoint 实现了 IO 多路复用(同步非阻塞)，NIO2Endpoint 实现了 IO 异步非阻塞。

### IO 多路复用(同步非阻塞)

IO 多路复用(同步非阻塞) 一般是用 selector 在一个无限循环内实现的。IO 多路复用含义是，一个 selector 可以询问多个连接的 I/O 事件是否就绪。线程获取请求数据分为两步:

1. 第一步，通过操作系统提供的 select 询问操作系统内核，某个连接的 IO 事件是否已就绪。
2. 第二步，若 IO 事件已就绪，通过操作系统提供的 read 命令，让操作系统内核把网卡的数据拷贝到用户空间下。如果网卡没有数据，这个 read 命令阻塞住，一直到有数据时才返回。由于第一步已经询问过操作系统内核，调用 read 命令时网卡有数据，因此达到了非阻塞的效果。

### 异步非阻塞

用户线程发起 read 请求内核读取数据时，会同时传入一个回调函数，此时线程不用等待。当数据到达网卡后，内核会自动把数据拷贝到用户空间下并调用回调函数。



## NioEndpoint

### NioEndpoint 职责

1. 负责接收请求连接和数据，并封装成 SocketProcessorBase，最后交给 Executor 线程池去执行。

### NioEndpoint 的结构

1. NioEndpoint  组件是  IO 多路复用模型的一种实现，包含 Acceptor, Poller, LimitLatch, SocketProcessorBase, Executor 共 5 个组件。主体结构 Acceptor + Poller + Executor 线程池是典型的主从 reactor 模式。
2. Acceptor 为主 reactor，负责监听连接。
3. Poller 为从 reactor，负责处理连接并借助 selector 监听连接中的 IO 事件。有 I/O 事件就绪后，将对应的 socket 连接封装成 SocketProcessorBase，交给 Executor 线程池处理。
4. Executor: 真正处理连接的线程池。
5. SocketProcessorBase: 具体 socket 的封装。
6. LimitLatch 负责限制请求连接数目。



## Acceptor

### Acceptor# run 方法

org.apache.tomcat.util.net.Acceptor#run

acceptor 使用独立的线程，用于接收 socket 并向后传递。注意这里接收的 socket 只是连接，不是具体的请求数据。

1. 一个独立的线程，无限循环。
2. 无限循环开头是边界条件处理，用于停止时结束循环。
3. 如果已经达到最大连接，则等待直到有空闲连接数。
4. 从 endpoint 获取 socket，是否阻塞要看具体实现。
   1. 注意这里阻塞不影响"NIO 非阻塞"，因为"NIO 非阻塞"指的是处理请求的线程不用阻塞地等待获取请求。
   2. 这里是 acceptor 线程，acceptor 线程专门用于接收请求并向后传递，acceptor 线程是否阻塞与"NIO 非阻塞"无关。
5. 调用 setSocketOptions 方法，将获取到的 socket 放进事件队列。

```java
public class Acceptor<U> implements Runnable {
    /**
     * acceptor 使用独立的线程，用于接收 socket 并向后传递。注意这里接收的 socket 只是连接，不是具体的请求数据。
     */
	@Override
    public void run() {
        int errorDelay = 0;
        long pauseStart = 0;

        try {
            // Loop until we receive a shutdown command
            // 如果 endpoint 处于运行中，无限循环
            while (!stopCalled) {
                // 如果 endpoint 处于暂停状态，则无限睡眠
                while (endpoint.isPaused() && !stopCalled) {
                    if (state != AcceptorState.PAUSED) {
                        pauseStart = System.nanoTime();
                        // Entered pause state
                        state = AcceptorState.PAUSED;
                    }
                    // ...
                }

                // 如果 endpoint 已经停止则退出无限循环
                if (stopCalled) {
                    break;
                }
                state = AcceptorState.RUNNING;
                
                try {
                    //if we have reached max connections, wait
                    // 如果已经达到最大连接，则等待直到有空闲连接数
                    endpoint.countUpOrAwaitConnection();
                    // Endpoint might have been paused while waiting for latch
                    // If that is the case, don't accept new connections
                    if (endpoint.isPaused()) {
                        continue;
                    }

                    U socket = null;
                    try {
                        /*
                         * Accept the next incoming connection from the server socket
                         *
                         * 从 endpoint 获取 socket，是否阻塞要看具体实现。
                         * 注意这里阻塞不影响"NIO 非阻塞"，因为"NIO 非阻塞"指的是处理请求的线程不用阻塞地等待获取请求。
                         * 这里是 acceptor 线程，acceptor 线程专门用于接收请求并向后传递，acceptor 线程是否阻塞与"NIO 非阻塞"无关。
                         */
                        socket = endpoint.serverSocketAccept();
                    } catch (Exception ioe) {
                        // ...
                    }
                    // Successful accept, reset the error delay
                    errorDelay = 0;

                    // Configure the socket
                    // endpoint 正常运行
                    if (!stopCalled && !endpoint.isPaused()) {
                        // setSocketOptions() will hand the socket off to
                        // an appropriate processor if successful
                        // 调用 setSocketOptions 方法，将获取到的 socket 传递至 endpoint
                        if (!endpoint.setSocketOptions(socket)) {
                            endpoint.closeSocket(socket);
                        }
                    } else {
                        // endpoint 未正常运行，则销毁 socket
                        endpoint.destroySocket(socket);
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    // ...
                }
            }
        } finally {
            stopLatch.countDown();
        }
        state = AcceptorState.ENDED;
    }
}
```

### NioEndpoint#serverSocketAccept 方法

org.apache.tomcat.util.net.NioEndpoint#serverSocketAccept

从 endpoint 获取 socket，NioEndpoint 中是一个阻塞版的实现。

1. 调用已经绑定过地址的 serverSocket 去获取 socket。
2. serverSock.accept() 是一个阻塞的方法，获取到 socket 时才会返回，未获取到则一直阻塞。具体实现是 ServerSocketChannelImpl#accept。

```java
    @Override
    protected SocketChannel serverSocketAccept() throws Exception {
        // 调用已经绑定过地址的 serverSocket 去获取 socket
        // 这是一个阻塞的方法，获取到 socket 时才会返回，未获取到则一直阻塞
        SocketChannel result = serverSock.accept();

        // Bug does not affect Windows. Skip the check on that platform.
        if (!JrePlatform.IS_WINDOWS) {
            SocketAddress currentRemoteAddress = result.getRemoteAddress();
            long currentNanoTime = System.nanoTime();
            if (currentRemoteAddress.equals(previousAcceptedSocketRemoteAddress) &&
                    currentNanoTime - previousAcceptedSocketNanoTime < 1000) {
                throw new IOException(sm.getString("endpoint.err.duplicateAccept"));
            }
            previousAcceptedSocketRemoteAddress = currentRemoteAddress;
            previousAcceptedSocketNanoTime = currentNanoTime;
        }

        return result;
    }
```

### ServerSocketChannelImpl#accept() 方法

sun.nio.ch.ServerSocketChannelImpl#accept()

获取绑定地址收到的 socket，如果有则返回，没有则无限循环监听地址是否有新的 socket，无限循环成阻塞效果。注意这里接收的 socket 只是连接，不是具体的请求数据。

```java
class ServerSocketChannelImpl extends ServerSocketChannel
        implements SelChImpl {
    private static NativeDispatcher nd;
    private final FileDescriptor fd;
    ServerSocket socket;
    public SocketChannel accept() throws IOException {
        synchronized (lock) {
            // 判断绑定的地址端口是否可用
            if (!isOpen())
                throw new ClosedChannelException();
            if (!isBound())
                throw new NotYetBoundException();
            SocketChannel sc = null;
    
            int n = 0;
            FileDescriptor newfd = new FileDescriptor();
            InetSocketAddress[] isaa = new InetSocketAddress[1];
            try {
                // 开始获取 socket
                begin();
                if (!isOpen())
                    return null;
                thread = NativeThread.current();
                for (;;) {
                    // 轮询是否有新的 socket，如果没有继续循环
                    // 如果有则 break 退出
                    n = accept(this.fd, newfd, isaa);
                    if ((n == IOStatus.INTERRUPTED) && isOpen())
                        continue;
                    break;
                }
            } finally {
                // ....
            }
            if (n < 1)
                return null;
            IOUtil.configureBlocking(newfd, true);
            InetSocketAddress isa = isaa[0];
            // 创建新的 socket 对象
            sc = new SocketChannelImpl(provider(), newfd, isa);
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                // ...
            return sc;
        }
    }
}
```

### NioEndpoint#setSocketOptions 方法

org.apache.tomcat.util.net.NioEndpoint#setSocketOptions

将获取到的 socket 传递至 endpoint

1. 根据是否启用 SSL 协议，从而确定具体使用哪个 NIOChannel 类型。
2. 放进 poller 事件队列前封装 socket，tomcat 线程池执行的 socket 对象的封装就是这个数据结构。
3. 把 socket 注册进 poller，里面会被转换成 PollerEvent 事件，并放进事件队列。

```java
    /**
     * Process the specified connection.
     *
     * 将获取到的 socket 传递至 endpoint
     *
     * @param socket The socket channel
     * @return <code>true</code> if the socket was correctly configured
     *  and processing may continue, <code>false</code> if the socket needs to be
     *  close immediately
     */
    @Override
    protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            // Allocate channel and wrapper
            // 根据是否启用 SSL 协议，从而确定具体使用哪个 NIOChannel 类型
            NioChannel channel = null;
            if (nioChannels != null) {
                channel = nioChannels.pop();
            }
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(bufhandler, this);
                } else {
                    channel = new NioChannel(bufhandler);
                }
            }
            // 放进 poller 事件队列前封装 socket，tomcat 线程池执行的 socket 对象的封装就是这个数据结构。
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            // Set socket properties
            // Disable blocking, polling will be used
            socket.configureBlocking(false);
            socketProperties.setProperties(socket.socket());

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            // 把 socket 注册进 poller，里面会被转换成 PollerEvent 事件，并放进事件队列
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            try {
                log.error(sm.getString("endpoint.socketOptionsError"), t);
            } catch (Throwable tt) {
                ExceptionUtils.handleThrowable(tt);
            }
            if (socketWrapper == null) {
                destroySocket(socket);
            }
        }
        // Tell to close the socket if needed
        return false;
    }
```



## NioEndpoint.Poller

### NioEndpoint.Poller#register 方法

org.apache.tomcat.util.net.NioEndpoint.Poller#register

把 socket 向 poller 注册。

1. 把 socket 封装成 pollerEvent。
2. pollerEvent 添加进事件阻塞队列。

```java
        /**
         * Registers a newly created socket with the poller.
         *
         * 把 socket 向 poller 注册
         *
         * @param socketWrapper The socket wrapper
         */
        public void register(final NioSocketWrapper socketWrapper) {
            socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
            // 把 socket 封装成 pollerEvent
            PollerEvent pollerEvent = createPollerEvent(socketWrapper, OP_REGISTER);
            // pollerEvent 添加进事件阻塞队列
            addEvent(pollerEvent);
        }
```

### NioEndpoint.Poller#addEvent 方法

org.apache.tomcat.util.net.NioEndpoint.Poller#addEvent

PollerEvent 添加进事件阻塞队列。

```java
        private void addEvent(PollerEvent event) {
            // PollerEvent 添加进事件阻塞队列
            events.offer(event);
            if (wakeupCounter.incrementAndGet() == 0) {
                selector.wakeup();
            }
        }
```

### NioEndpoint.Poller#run 方法

Poller 是 NioEndpoint 的内部类。

1. poller 的无限循环，在 NioEndpoint#startInternal 方法中会启动 poller 线程，一直执行到 Poller#destroy() 销毁方法被调用。
2. 通过 selector 检查 socket(socketChannel) 中是否有数据并获取数据，然后调用 processKey 方法处理 socket。
3. selector 只是一个工具，没有自己独立的线程，通过 selector 获取 socket 使用的是 poller 的独立线程。
4. Poller#run 方法里还有处理超时 SocketChannel 的逻辑。

执行流程:

1. 如果 poller 没有关闭，则获取 event 事件。如果 poller 关闭了，则最后获取一次事件，以防止还有事件没有读取。
2. selector#selectNow 和 select 方法，调用操作系统 IO 函数(例如 select, poll, epoll 等)，向内核查询 SocketChannel 的状态是否有新数据。具体使用何种取决于操作系统，linux 一般使用性能较好的 epoll。
3. 如果之前 selector 调用操作系统 IO 函数发现有新数据，则从 selector 中获取 selectionKey。selectionKey 里面封装了 socket，socket 是在 Poller#events 方法中关联到 selector 的。
4. 获取 selectionKey 对应的 socketWrapper，执行 processKey 方法，处理 socket。
5. 循环遍历检查自己所管理的 SocketChannel 是否已经超时，如果有超时就关闭这个 SocketChannel。

```java
        /**
         * The background thread that adds sockets to the Poller, checks the
         * poller for triggered events and hands the associated socket off to an
         * appropriate processor as events occur.
         *
         * poller 的无限循环，在 NioEndpoint#startInternal 方法中会启动 poller 线程，一直执行到 Poller#destroy() 销毁方法被调用
         * 通过 selector 检查 socket(socketChannel) 中是否有数据并获取数据，然后调用 processKey 方法处理 socket。
         * selector 只是一个工具，没有自己独立的线程，通过 selector 获取 socket 使用的是 poller 的独立线程。
         * Poller#run 方法里还有处理超时 SocketChannel 的逻辑
         */
        @Override
        public void run() {
            // Loop until destroy() is called
            while (true) {
                // Poller 不断的通过内部的 Selector 对象向内核查询 Channel 的状态，一旦可读就生成任务类 SocketProcessor 交给 Executor 去处理。

                boolean hasEvents = false;

                try {
                    // 如果 poller 没有关闭，则获取 socket
                    if (!close) {
                        // events
                        hasEvents = events();
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            /*
                             * If we are here, means we have other stuff to do
                             * Do a non blocking select
                             *
                             * selector#selectNow 和 select 方法，调用操作系统 IO 函数(例如 select, poll, epoll 等)，向内核查询 SocketChannel 的状态是否有新数据。
                             * 具体使用何种取决于操作系统，linux 一般使用性能较好的 epoll
                             */
                            keyCount = selector.selectNow();
                        } else {
                            keyCount = selector.select(selectorTimeout);
                        }
                        wakeupCounter.set(0);
                    }
                    // 如果 poller 关闭了，则最后获取一次 PollerEvent 事件，以防止还有事件没有读取
                    if (close) {
                        events();
                        timeout(0, false);
                        try {
                            // poller 关闭时把 selector 关闭
                            selector.close();
                        } catch (IOException ioe) {
                            log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                        }
                        break;
                    }
                    // Either we timed out or we woke up, process events first
                    if (keyCount == 0) {
                        hasEvents = (hasEvents | events());
                    }
                } catch (Throwable x) {
                    ExceptionUtils.handleThrowable(x);
                    log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
                    continue;
                }

                // 如果之前 selector 调用操作系统 IO 函数发现有新数据，则从 selector 中获取 selectionKey
                // selectionKey 里面封装了 socket，socket 是在 Poller#events 方法中关联到 selector 的。
                Iterator<SelectionKey> iterator =
                    keyCount > 0 ? selector.selectedKeys().iterator() : null;
                // Walk through the collection of ready keys and dispatch
                // any active event.
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    iterator.remove();
                    // 获取 selectionKey 对应的 socketWrapper
                    NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                    // Attachment may be null if another thread has called
                    // cancelledKey()
                    if (socketWrapper != null) {
                        // 执行 processKey 方法，处理 socket
                        processKey(sk, socketWrapper);
                    }
                }

                // Process timeouts
                // poller 另一个重要的任务，循环遍历检查自己所管理的 SocketChannel 是否已经超时，如果有超时就关闭这个 SocketChannel
                timeout(keyCount,hasEvents);
            }

            // latch 锁数量 -1
            getStopLatch().countDown();
        }
```

### NioEndpoint.Poller#events 方法

org.apache.tomcat.util.net.NioEndpoint.Poller#events

是否有 PollerEvent 待处理，PollerEvent 里面封装了接收到的 socket。从事件阻塞队列里获取 PollerEvent，并把其中的 socket 与 selector 关联并生成 SelectionKey，SelectionKey 里封装了 socket。

```java
        /**
         * Processes events in the event queue of the Poller.
         *
         * 是否有 PollerEvent 待处理，PollerEvent 里面封装了接收到的 socket
         * 从事件阻塞队列里获取 PollerEvent，并把其中的 socket 与 selector 关联并生成 SelectionKey，SelectionKey 里封装了 socket。
         *
         * @return <code>true</code> if some events were processed,
         *   <code>false</code> if queue was empty
         */
        public boolean events() {
            boolean result = false;

            PollerEvent pe = null;
            for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
                result = true;
                NioSocketWrapper socketWrapper = pe.getSocketWrapper();
                SocketChannel sc = socketWrapper.getSocket().getIOChannel();
                int interestOps = pe.getInterestOps();
                if (sc == null) {
                    log.warn(sm.getString("endpoint.nio.nullSocketChannel"));
                    socketWrapper.close();
                } else if (interestOps == OP_REGISTER) {
                    try {
                        // 把 socket 注册进 selector
                        sc.register(getSelector(), SelectionKey.OP_READ, socketWrapper);
                    } catch (Exception x) {
                        log.error(sm.getString("endpoint.nio.registerFail"), x);
                    }
                } else {
                    // 把 socket 与 selector 关联并得到 SelectionKey，SelectionKey 里封装了 socket
                    final SelectionKey key = sc.keyFor(getSelector());
                    if (key == null) {
                        socketWrapper.close();
                    } else {
                        final NioSocketWrapper attachment = (NioSocketWrapper) key.attachment();
                        if (attachment != null) {
                            // We are registering the key to start with, reset the fairness counter.
                            try {
                                int ops = key.interestOps() | interestOps;
                                attachment.interestOps(ops);
                                key.interestOps(ops);
                            } catch (CancelledKeyException ckx) {
                                cancelledKey(key, socketWrapper);
                            }
                        } else {
                            cancelledKey(key, socketWrapper);
                        }
                    }
                }
                if (running && eventCache != null) {
                    pe.reset();
                    // 把 PollerEvent 添加进缓存
                    eventCache.push(pe);
                }
            }
            return result;
        }
```

### Selector

Selector 使用的操作系统 IO 函数不只 select，包含 select, poll, epoll 等，根据操作系统决定使用何种函数，如果操作系统支持，通常使用性能更好的 epoll。

### NioEndpoint.Poller#processKey 方法

org.apache.tomcat.util.net.NioEndpoint.Poller#processKey

处理 selectionKey(封装了 socket)。

1. 如果 poller 关闭，则取消对 selectionKey 的处理。
2. 分开处理文件类型 socket，处理可读类型 socket，处理可写类型 socket。对于读写类型 socket，调用 AbstractEndpoint#processSocket 方法处理。

```java
        /**
         * 处理 socket
         */
        protected void processKey(SelectionKey sk, NioSocketWrapper socketWrapper) {
            try {
                // 如果 poller 关闭，则取消对 selectionKey 的处理
                if (close) {
                    cancelledKey(sk, socketWrapper);
                } else if (sk.isValid()) {
                    if (sk.isReadable() || sk.isWritable()) {
                        // 处理文件类型 socket
                        if (socketWrapper.getSendfileData() != null) {
                            processSendfile(sk, socketWrapper, false);
                        } else {
                            unreg(sk, socketWrapper, sk.readyOps());
                            boolean closeSocket = false;
                            // Read goes before write
                            // 处理可读类型 socket
                            if (sk.isReadable()) {
                                if (socketWrapper.readOperation != null) {
                                    if (!socketWrapper.readOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.readBlocking) {
                                    synchronized (socketWrapper.readLock) {
                                        socketWrapper.readBlocking = false;
                                        socketWrapper.readLock.notify();
                                    }
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_READ, true)) {
                                    closeSocket = true;
                                }
                            }
                            // 处理可写类型 socket
                            if (!closeSocket && sk.isWritable()) {
                                if (socketWrapper.writeOperation != null) {
                                    if (!socketWrapper.writeOperation.process()) {
                                        closeSocket = true;
                                    }
                                } else if (socketWrapper.writeBlocking) {
                                    synchronized (socketWrapper.writeLock) {
                                        socketWrapper.writeBlocking = false;
                                        socketWrapper.writeLock.notify();
                                    }
                                } else if (!processSocket(socketWrapper, SocketEvent.OPEN_WRITE, true)) {
                                    closeSocket = true;
                                }
                            }
                            if (closeSocket) {
                                cancelledKey(sk, socketWrapper);
                            }
                        }
                    }
                } else {
                    // Invalid key
                    cancelledKey(sk, socketWrapper);
                }
            } catch (CancelledKeyException ckx) {
                cancelledKey(sk, socketWrapper);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("endpoint.nio.keyProcessingError"), t);
            }
        }
```



## AbstractEndpoint#processSocket 方法

org.apache.tomcat.util.net.AbstractEndpoint#processSockett

处理 socket。

1. 根据 socket 和 SocketEvent 获得 socketProcessorBase。
2. 使用线程池执行 socketProcessorBase，线程池的来源请见 AbstractEndpoint#createExecutor 方法。

```java
    public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = null;
            if (processorCache != null) {
                sc = processorCache.pop();
            }
            if (sc == null) {
                // socketProcessorBase 根据 socket 和 SocketEvent 获得
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            // 使用线程池执行 socketProcessorBase，线程池的来源请见 AbstractEndpoint#createExecutor 方法
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } catch (RejectedExecutionException ree) {
            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
            return false;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            // This means we got an OOM or similar creating a thread, or that
            // the pool and its queue are full
            getLog().error(sm.getString("endpoint.process.fail"), t);
            return false;
        }
        return true;
    }
```

### SocketProcessorBase#run 方法

org.apache.tomcat.util.net.SocketProcessorBase#run

1. SocketProcessorBase 实现了 Runnable 接口，对于 socket 的具体处理会在这里面进行。实现很简单，加 socket 维度的锁，然后调用子类实现的 doRun 方法做处理。

```java
public abstract class SocketProcessorBase<S> implements Runnable {
    protected SocketWrapperBase<S> socketWrapper;
    protected SocketEvent event;

	@Override
    public final void run() {
        // 加 socket 维度的锁
        synchronized (socketWrapper) {
            // It is possible that processing may be triggered for read and
            // write at the same time. The sync above makes sure that processing
            // does not occur in parallel. The test below ensures that if the
            // first event to be processed results in the socket being closed,
            // the subsequent events are not processed.
            if (socketWrapper.isClosed()) {
                return;
            }
            // 对 socket 具体的处理
            doRun();
        }
    }
}
```

### NioEndpoint.SocketProcessor#doRun 方法

org.apache.tomcat.util.net.NioEndpoint.SocketProcessor#doRun

socket 握手(经典的 TCP 三次握手)相关逻辑，握手正常则传递给 handler(Processor) 进行相关处理。

```java
        @Override
        protected void doRun() {
            Poller poller = NioEndpoint.this.poller;
            if (poller == null) {
                socketWrapper.close();
                return;
            }

            try {
                // socket 握手(经典的 TCP 三次握手)相关逻辑
                int handshake = -1;
                try {
                    if (socketWrapper.getSocket().isHandshakeComplete()) {
                        // No TLS handshaking required. Let the handler
                        // process this socket / event combination.
                        handshake = 0;
                    } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                            event == SocketEvent.ERROR) {
                        // Unable to complete the TLS handshake. Treat it as
                        // if the handshake failed.
                        handshake = -1;
                    } else {
                        handshake = socketWrapper.getSocket().handshake(event == SocketEvent.OPEN_READ, event == SocketEvent.OPEN_WRITE);
                        event = SocketEvent.OPEN_READ;
                    }
                } catch (IOException x) {
                    // ...
                } catch (CancelledKeyException ckx) {
                    handshake = -1;
                }
                // 握手正常，传递给 handler 进行相关处理。connector 模块的职责结束，后续流程是 handler 负责。
                if (handshake == 0) {
                    SocketState state = SocketState.OPEN;
                    // Process the request from this socket
                    if (event == null) {
                        state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
                    } else {
                        state = getHandler().process(socketWrapper, event);
                    }
                    if (state == SocketState.CLOSED) {
                        poller.cancelledKey(getSelectionKey(), socketWrapper);
                    }
                    // ...
            } catch (CancelledKeyException cx) {
                poller.cancelledKey(getSelectionKey(), socketWrapper);
            } catch (VirtualMachineError vme) {
                ExceptionUtils.handleThrowable(vme);
            } catch (Throwable t) {
                log.error(sm.getString("endpoint.processing.fail"), t);
                poller.cancelledKey(getSelectionKey(), socketWrapper);
            } finally {
                socketWrapper = null;
                event = null;
                // return to cache
                if (running && processorCache != null) {
                    processorCache.push(this);
                }
            }
        }
```



## Processor 接口

org.apache.coyote.Processor

重点注意 process, getRequest 和 recycle 方法。

```java
/**
 * Common interface for processors of all protocols.
 */
public interface Processor {
    SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status) throws IOException;

    UpgradeToken getUpgradeToken();

    boolean isUpgrade();
    
    boolean isAsync();

    void timeoutAsync(long now);

    Request getRequest();

    void recycle();

    void setSslSupport(SSLSupport sslSupport);

    ByteBuffer getLeftoverInput();

    void pause();

    boolean checkAsyncTimeoutGeneration();
}
```



## AbstractProcessorLight#process 方法

org.apache.coyote.AbstractProcessorLight#process

模板方法，在父类中维护一个 SocketState 状态机，根据状态执行对应方法。http 请求和 service 方法相关，而 websocket 和 dispatch 方法相关。

1. do while 进入 SocketState 状态机。
2. webSocket 才会用到 dispatch 的逻辑，对于 http 请求无需关心。
3. AbstractProcessorLight#service 方法的子类实现处理 http 请求。

```java
    /**
     * 模板方法，在父类中维护一个 SocketState 状态机，根据状态执行对应方法。http 请求和 service 方法相关，而 websocket 和 dispatch 方法相关。
     */
	@Override
    public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
            throws IOException {

        SocketState state = SocketState.CLOSED;
        Iterator<DispatchType> dispatches = null;
        // do while 进入 SocketState 状态机
        do {
            if (dispatches != null) {
                // ...
                // webSocket 才会用到 dispatch 的逻辑，对于 http 请求无需关心
                state = dispatch(nextDispatch.getSocketStatus());
                if (!dispatches.hasNext()) {
                    state = checkForPipelinedData(state, socketWrapper);
                }
            } else if (status == SocketEvent.DISCONNECT) {
                // Do nothing here, just wait for it to get recycled
            } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                state = dispatch(status);
                state = checkForPipelinedData(state, socketWrapper);
            } else if (status == SocketEvent.OPEN_WRITE) {
                // Extra write event likely after async, ignore
                state = SocketState.LONG;
            } else if (status == SocketEvent.OPEN_READ) {
                // AbstractProcessorLight#service 方法的子类实现处理 http 请求
                state = service(socketWrapper);
            } else if (status == SocketEvent.CONNECT_FAIL) {
                logAccess(socketWrapper);
            } else {
                // Default to closing the socket if the SocketEvent passed in
                // is not consistent with the current state of the Processor
                state = SocketState.CLOSED;
            }
            // ...
        } while (state == SocketState.ASYNC_END ||
                dispatches != null && state != SocketState.CLOSED);

        return state;
    }
```



## Http11Processor#service 方法

org.apache.coyote.http11.Http11Processor#service

http1.1 协议处理 http 请求。

1. 初始化一些参数，验证设置 request，SocketState 判断返回等流程。
2. 获得 adapter 实例，调用 Adapter#service 方法。这里的 adapter 对象实例是 CoyoteAdapter 类型。

```java
	@Override
    public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
        // 初始化一些参数，略过
        // ...

        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                sendfileState == SendfileState.DONE && !endpoint.isPaused()) {
            // Parsing the request header
            // 验证设置 request 等流程略过
            // ...

            // Process the request in the adapter
            if (getErrorState().isIoAllowed()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    // 获得 adapter 实例，调用 Adapter#service 方法。这里的 adapter 对象实例是 CoyoteAdapter 类型。
                    getAdapter().service(request, response);
                    if(keepAlive && !getErrorState().isError() && !isAsync() &&
                            statusDropsConnection(response.getStatus())) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                } catch (InterruptedIOException e) {
                    setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
                } catch (HeadersTooLargeException e) {
                    log.error(sm.getString("http11processor.request.process"), e);
                    // The response should not have been committed but check it
                    // anyway to be safe
                    if (response.isCommitted()) {
                        setErrorState(ErrorState.CLOSE_NOW, e);
                    } else {
                        response.reset();
                        response.setStatus(500);
                        setErrorState(ErrorState.CLOSE_CLEAN, e);
                        response.setHeader("Connection", "close"); // TODO: Remove
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("http11processor.request.process"), t);
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }
            // SocketState 判断略过
            // ...
            sendfileState = processSendfile(socketWrapper);
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

        // SocketState 返回略过
        // ...
    }
```



## CoyoteAdapter#service 方法

org.apache.catalina.connector.CoyoteAdapter#service

1. CoyoteAdapter 适配器，适配了 connector 和 container，使 Connector 能调用进 Container。具体是调用了 pipeline() 中 valve 的 invoke 方法。至此 connector 模块调用进了 container，后续是 tomcat-container 的内容。
2. 在这里面也完成了Request 和 Response 的类型转换，将 connector 模块中的 Request/Response 转换成了 container 中的。

```java
	@Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {
        // 将 org.apache.coyote 包下的 Request 和 Response, 转换成 org.apache.catalina.connector 包下的类型
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }
        
        // ...
        
        try {
            // Parse and set Catalina and configuration specific
            // request parameters
            // 解析并设置 Catalina 核心属性和特定请求配置，如 host, context。根据请求路径来判断由哪个 StandardWrapper 处理这个请求也是在这里面完成的。
            postParseSuccess = postParseRequest(req, request, res, response);
            if (postParseSuccess) {
                //check valves if we support async
                request.setAsyncSupported(
                        connector.getService().getContainer().getPipeline().isAsyncSupported());
                // Calling the container
                // 调用进 Container 中，这里是关键，由 Connector 模块调用进了 Container，完成两者的适配
                connector.getService().getContainer().getPipeline().getFirst().invoke(
                        request, response);
            }
            // ...
        } catch (IOException e) {
            // Ignore
        } finally {
            // ...
        }
    }
```



## Handler

todo，待补充。
