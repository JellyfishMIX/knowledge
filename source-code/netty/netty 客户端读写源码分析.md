# netty 读写源码分析



## 说明

1. 本文基于 netty 4.1 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 客户端写入

1. 左下角客户端，把要发送的数据放入一个缓冲链表，然后拿缓冲列表中元素做具体的数据发送。

![Netty 客户端向服务端发送请求的过程.png](https://image-hosting.jellyfishmix.com/20230703171436.png)

### AbstractChannel.AbstractUnsafe#write 方法 -- 写入数据

io.netty.channel.AbstractChannel.AbstractUnsafe#write

主要是给 ChannelOutboundBuffer 添加一个 message。

```java
        @Override
        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                try {
                    // release message now to prevent resource-leak
                    ReferenceCountUtil.release(msg);
                } finally {
                    safeSetFailure(promise,
                            newClosedChannelException(initialCloseCause, "write(Object, ChannelPromise)"));
                }
                return;
            }

            int size;
            try {
                msg = filterOutboundMessage(msg);
                size = pipeline.estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                try {
                    ReferenceCountUtil.release(msg);
                } finally {
                    safeSetFailure(promise, t);
                }
                return;
            }

            // 给 ChannelOutboundBuffer 添加一个 message
            outboundBuffer.addMessage(msg, size, promise);
        }
```

### ChannelOutboundBuffer#addMessage 方法 -- 添加一个 message

io.netty.channel.ChannelOutboundBuffer#addMessage

1. 新建一个 entry 加入 unflushedEntry。

```java
    public void addMessage(Object msg, int size, ChannelPromise promise) {
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        // 新建一个 entry 入队
        if (tailEntry == null) {
            flushedEntry = null;
        } else {
            Entry tail = tailEntry;
            tail.next = entry;
        }
        tailEntry = entry;
        // 加入 unflushedEntry
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }

        // increment pending bytes after adding message to the unflushed arrays.
        // See https://github.com/netty/netty/issues/1619
        incrementPendingOutboundBytes(entry.pendingSize, false);
    }
```

### ChannelOutboundBuffer#addFlush 方法 -- 链表元素转移

io.netty.channel.ChannelOutboundBuffer#addFlush

把 unflushedEntry 链表中的元素转移至 flushedEntry 链表中。

```java
    public void addFlush() {
        // 取出 unflushedEntry
        Entry entry = unflushedEntry;
        if (entry != null) {
            if (flushedEntry == null) {
                // there is no flushedEntry yet, so start with the entry
                // 把需要发送的数据放入 flushedEntry 中
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    // Was cancelled so make sure we free up memory and notify about the freed bytes
                    int pending = entry.cancel();
                    decrementPendingOutboundBytes(pending, false, true);
                }
                entry = entry.next;
            } while (entry != null);

            // All flushed so reset unflushedEntry
            unflushedEntry = null;
        }
    }
```

### NioSocketChannel#doWrite 方法 -- 向 SocketChannel 中写入数据

io.netty.channel.socket.nio.NioSocketChannel#doWrite

1. 把 buffer 数据向 SocketChannel 中写入

```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        SocketChannel ch = javaChannel();
        int writeSpinCount = config().getWriteSpinCount();
        do {
            if (in.isEmpty()) {
                // All written so clear OP_WRITE
                clearOpWrite();
                // Directly return here so incompleteWrite(...) is not called.
                return;
            }

            // Ensure the pending writes are made of ByteBufs only.
            int maxBytesPerGatheringWrite = ((NioSocketChannelConfig) config).getMaxBytesPerGatheringWrite();
            ByteBuffer[] nioBuffers = in.nioBuffers(1024, maxBytesPerGatheringWrite);
            int nioBufferCnt = in.nioBufferCount();

            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    writeSpinCount -= doWrite0(in);
                    break;
                case 1: {
                    ByteBuffer buffer = nioBuffers[0];
                    int attemptedBytes = buffer.remaining();
                    // 把 buffer 数据向 SocketChannel 中写入
                    final int localWrittenBytes = ch.write(buffer);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    adjustMaxBytesPerGatheringWrite(attemptedBytes, localWrittenBytes, maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
                default: {
                    long attemptedBytes = in.nioBufferSize();
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                    if (localWrittenBytes <= 0) {
                        incompleteWrite(true);
                        return;
                    }
                    // Casting to int is safe because we limit the total amount of data in the nioBuffers to int above.
                    adjustMaxBytesPerGatheringWrite((int) attemptedBytes, (int) localWrittenBytes,
                            maxBytesPerGatheringWrite);
                    in.removeBytes(localWrittenBytes);
                    --writeSpinCount;
                    break;
                }
            }
        } while (writeSpinCount > 0);

        incompleteWrite(writeSpinCount < 0);
    }
```



## 客户端接收服务端发来的响应

1. 服务端 NioEventLoop 线程把响应数据写入 SocketChannel 中，发送给客户端。客户端 NioEventLoop 线程持有的 Selector 监听到 OP_READ 事件后，会把响应数据读出来，然后调用用户自定义 ChannelHandler 来处理响应数据。

![Netty 客户端如何接收服务端发送的相应.png](https://image-hosting.jellyfishmix.com/20230703171929.png)

### AbstractChannelHandlerContext#invokeChannelRead 方法 -- 调用 ChannelHandler 处理收到的数据

io.netty.channel.AbstractChannelHandlerContext#invokeChannelRead(java.lang.Object)

```java
    private void invokeChannelRead(Object msg) {
        if (invokeHandler()) {
            try {
                final ChannelHandler handler = handler();
                final DefaultChannelPipeline.HeadContext headContext = pipeline.head;
                if (handler == headContext) {
                    // 自定义 ChannelHandler 会在这里进行处理
                    headContext.channelRead(this, msg);
                } else if (handler instanceof ChannelDuplexHandler) {
                    ((ChannelDuplexHandler) handler).channelRead(this, msg);
                } else {
                    ((ChannelInboundHandler) handler).channelRead(this, msg);
                }
            } catch (Throwable t) {
                invokeExceptionCaught(t);
            }
        } else {
            fireChannelRead(msg);
        }
    }
```

