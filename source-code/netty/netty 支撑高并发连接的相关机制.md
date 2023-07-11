# netty 支撑高并发连接的相关机制



## 说明

1. 本文基于 netty 4.1 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## netty NIO 机制

### 流程

1. 服务端有三组线程池，BossGroup 线程池，WorkerGroup 线程池，业务线程池。
2. BossGroup 线程池，这个线程池的功能是负责客户端的连接请求，一般情况仅为一个线程。每一个线程都封装为 EventLoop，服务端在启动时，BossGroup 线程组内的每个 EventLoop 都会创建一个 NioServerSocketChannel，用来监听服务端的端口，判断是否收到客户端连接请求。通过把 NioServerSocketChannel 注册到 EventLoop 持有的 Selector 上，关注 OP_ACCEPT(接收连接)事件，Selector 循环监听注册的 NioServerSocketChannel 的网络事件。
3. BossGroup 线程的 NioServerSocketChannel 收到 OP_ACCEPT(接收连接)事件后，建立连接，创建一个新的 SocketChannel，同时会初始化这个 SocketChannel 对应的 Pipeline，每个 SocketChannel 对应一个 Pipeline，每个 Pipeline 包含多个 ChannelHandler。
4. 新的 SocketChannel 创建后，会注册到 WorkerGroup 线程池的 EventLoop(即一个线程)持有的 Selector 上，关注 OP_READ(读)事件。一个 Selector 可以注册多个 SocketChannel，即一个 Selector 可以监听多个 SocketChannel 的网络事件。
5. 客户端向服务端发送请求时，WorkerGroup 中的 Selector 会监听到 OP_READ(读)事件，触发 Pipleline 责任链，Pipeline 中的入站 InboundHandler 会按顺序执行。具体处理读事件时使用业务线程池。

![Netty 线程模型.png](https://image-hosting.jellyfishmix.com/20230705112231.png)

### netty NIO 机制的优点

1. 通过一个线程持有的 Selector 同时监听多个连接的网络事件，即 IO 多路复用。相比 BIO 一个连接占用一个线程，大大节约了线程资源。
2. 主从 Reactor 模式，即监听 OP_ACCEPT(接收连接)事件、建立连接使用一个线程，监听 OP_READ(读) 事件使用另一个线程。线程资源隔离。
   1. 避免读事件太多处理负载重，导致客户端连接请求建立延迟，进而造成客户端连接请求的重试。
   2. 避免需要建立的连接太多，影响读事件的处理。
3. 每个线程池都可以根据负载扩容，灵活横向扩展。



## 缓冲区机制(内存池)

1. netty 对 java NIO 原生的 ByteBuffer 做了封装。

### java 原生 NIO ByteBuffer 的问题

1. NIO 的 ByteBuffer 在创建时设定了长度，而且在使用过程中无法扩容。
2. API 使用复杂。向 ByteBuffer 写完数据后要读数据时，需要调用 flip() 方法转变为读模式，改变 index 到写的起始位置，才能开始读取数据。读模式转换为写模式时需要调用 rewind() 方法。

### Netty 的 ByteBuf

### ByteBuf 的结构

1. ByteBuf 的结构主要有下面几个特点。
   1. ByteBuf 是一个 Byte 容器，用于操作和管理缓冲区。
   2. readerIndex 表示读操作可执行的位置，writerIndex 表示写操作执行的位置，capacity 表示 ByteBuf 最大的位置。readerIndex 只能增加到与 writerIndex 相等。类似地，writerIndex 也只能增加到与 capacity 相等。
   3. readerIndex 与 writerIndex 之间是可读的部分，writerIndex 与 capacity 之间是可写的部分。
   4. ByteBuf 分为三个部分: 已抛弃 bytes，可读 bytes，可写 bytes。
      1. 已抛弃 Bytes: 这部分表示已经执行完读操作的的 bytes，这部分大小会随着 readerIndex 的增加而增大。因为这部分读写都完成了，所以被称为已抛弃 bytes，等待着被回收。

![ByteBuf 基本结构.png](https://image-hosting.jellyfishmix.com/20230706125030.png)

### ByteBuf 三种内存分配方式

ByteBuf 按分配位置可分为三种缓冲区。

1. 堆缓冲: 在 JVM 堆内申请内存。
2. 直接缓冲区: JVM 堆外申请内存。
3. 复合缓冲区: 由请求头和请求体组成，一个 ByteBuf 专门放请求头，另一个专门放请求体。

### AbstractNioByteChannel.NioByteUnsafe#read

io.netty.channel.nio.AbstractNioByteChannel.NioByteUnsafe#read

```java
        @Override
        public final void read() {
            // 获取 channel 的配置信息
            final ChannelConfig config = config();
            if (shouldBreakReadReady(config)) {
                clearReadPending();
                return;
            }
            final ChannelPipeline pipeline = pipeline();
            // 获取 channel 的 buffer 分配器
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    // 把 socketChannel 中的数据读到计算好大小的 byteBuf 中
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    // 把网络事件传递给 pipeline 上下一个 InBoundHandler
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
    }
```

