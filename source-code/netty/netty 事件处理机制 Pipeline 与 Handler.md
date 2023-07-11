# netty 事件处理机制 Pipeline 与 Handler



## 说明

1. 本文基于 netty 4.1 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## netty 对 SocketChannel 的封装 -- Channel 接口

### netty 没有直接使用 NIO 原生的 SocketChannel 原因

1. NIO 对于 Channel 没有统一的接口，只有 SocketChannel 和 SeverSocketChannel 这两个类，因此如果不封装这两个类，要分别处理。
2. netty 的 Pipeline, EventLoop 以及负责处理网络 IO 事件的功能需要整合到 Channel 里，因此 netty 需要封装 Channel。
3. 自定义 Channel 接口更加灵活。

### Channel 接口

io.netty.channel.Channel

Channel 接口封装了所有的网络事件，同时继承了 AttributeMap，用于保存连接的一些属性。

![image.png](https://image-hosting.jellyfishmix.com/20230703214811.jpeg)

### AbstractChannel

io.netty.channel.AbstractChannel

Channel 的实现类里最重要的是抽象类 AbstractChannel

1. 分配 channelId。
2. 新建实际进行网络 IO 的对象 unsafe。
3. 新建此 channel 对应的 pipeline。

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        // 分配 channelId
        id = newId();
        // 实际进行网络 IO 的对象
        unsafe = newUnsafe();
        // 新建此 channel 对应的 pipeline
        pipeline = newChannelPipeline();
    }
```



## ChannelHandler

用于处理网络事件。

网络事件有多种，各种事件的执行顺序如下:

1. handlerAdd: ChannelHandler 对象加载到 Pipeline 的时候，此方法会被回调。
2. channelRegistered: Channel 对象注册到 EventLoop 持有的 Selector 上时，此方法会被回调。
3. channelActive: 通道被激活的时候，也就是 TCP 连接建立成功时，此方法会被回调。
4. channelRead: 读到了对方发来的数据时，此方法会被回调。
5. channelReadComplete: 读完对方发来的数据时，此方法会被回调。
6. channelInactive: 当连接发生问题，比如连接的另一方挂了，此方法会被回调。
7. channelUnregisgtered: 当 Channel 在 EventLoop 持有的 Selector 上被取消注册时，此方法会被回调。
8. handlerRemoved: 当 ChannelHandler 从 Pipeline 中删除，此方法会被回调。



## Pipeline

### 责任链模式

Pipeline 应用了责任链模式，每个 ChannelHandler 定义特定的功能，按一定顺序进行装配，使用时按顺序调用。优点:

1. 解耦。比起把所有功能都放入一个逻辑里，每个 ChannelHandler 只包含一个特定的功能，可以大大降低耦合性。
2. 灵活。可以方便地加减 ChannelHandler，甚至在运行时也可以从责任链增加或减少 ChannelHandler。

### 入站事件和出站事件

Pipeline 中的 ChannelHandler 按事件类型分为两类:

1. InboundHandler 入站事件: 表示由被动 IO 事件触发，进入到 pipline 责任链中，所以叫入站事件。主要包括读事件, 读完成事件, 注册事件, 解除注册事件, 活跃事件, 非活跃事件等。
2. OutboundHandler 出站事件: 表示由主动 IO 事件触发，从 pipeline 责任链中出去，所以叫出站事件。主要包括端口绑定事件, 连接事件, 写事件等这些主动触发的事件。

### 向 ChannelPipeline 添加 ChannelHandler

io.netty.channel.ChannelPipeline

不仅可以直接添加 ChannelHandler，而且可以通过线程池参数 EventExecutorGroup 异步执行 ChannelHandler 逻辑。避免某一个 ChannelHandler 长时间阻塞而拖延了整体 pipeline 的执行。

```java
ChannelPipeline addLast(String name, ChannelHandler handler);

ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler);

ChannelPipeline addLast(ChannelHandler... handlers);

ChannelPipeline addLast(EventExecutorGroup group, ChannelHandler... handlers);
```

### EventLoop, Selector, SocketChannel, Pipeline, ChannelHandler 的关系

1. 一个 EventLoop 持有一个 Selector，一个 Selector 可以监听和管理多个连接(SocketChannel)。
2. 一个 SocketChannel 对应一个 Pipeline。
3. 一个 Pipeline 中有多个 ChannelHandler 来处理各种网络事件。

![NioEventLoop,SocketChannel与Pipeline的关系.png](https://image-hosting.jellyfishmix.com/20230704155938.png)

### Pipeline 内 Hander 的执行顺序

1. 对于入站 InboundHandler, 按照 ChannelHandler 加入 Pipeline 的顺序执行。

![Pipeline 入站回调顺序.png](https://image-hosting.jellyfishmix.com/20230704165721.png)

2. 对于出站 OutboundHandler, 按照 ChannelHandler 加入 Pipeline 的倒序执行。

   ![Pipeline 出站回调顺序.png](https://image-hosting.jellyfishmix.com/20230704165840.png)



## ChannelHandlerContext

1. ChannelHandler 是无状态的，但是 Pipeline 需要有状态。ChannelHandlerContext 实现了 Pipeline 的状态化。下图描述了 Pipeline, ChannelHandlerContext 和 ChannelHandler 的关系。
2. ChannelHandler 本身是无状态的，加入 Pipeline 后，会被一个 ChannelHandlerContext 包裹使之有状态。ChannelHandlerContext 的功能:
   1. 可以获取上下文的 channel, eventLoop，对应的 Pipeline 等组件资源。
   2. 可以调用出站入站的方法，用于事件在不同的 Handler 之间传播。

![Handler与ChannelHandlerContext的关系.png](https://image-hosting.jellyfishmix.com/20230704170140.png)

ChannelHandlerContext 继承了入站调用器 ChannelInboundInvoker 和出站调用器 ChannelOutboundInvoker，分别定义了出站和入站的事件传播方法。

```java
public interface ChannelHandlerContext extends AttributeMap, ChannelInboundInvoker, ChannelOutboundInvoker {
}
```

### HeadContext 和 TailContext

1. 在 Channel 初始化时，netty 会加上两个 Context，分别是 HeadContext 和 TailContext。
2. 这两个 Context 分别在 Pipeline 的头部和尾部。用户自定义的 Context 在 HeadContext 和 TailContext 之间。
3. 出站事件和入站事件都会执行 HeadContext。不同的是:
   1. 入站事件第一个执行的是 HeadContext，最后执行的是 TailContext。
   2. 出站事件第一个执行的是 TailContext，最后执行的是 HeadContext。

![HeadContext 和 TailContext.png](https://image-hosting.jellyfishmix.com/20230704211352.png)



## 事件的传播

io.netty.channel.DefaultChannelPipeline

传播即事件的流转(传递)，Pipeline 中事件的传播方式可以从 DefaultChannelPipeline 中看到。

入站事件 read() 从 Pipeline 的头部往尾部传。出站事件 write() 从 Pipeline 的尾部往头部传。

```java
/**
 * 入站方法 read()
 */
@Override
public final ChannelPipeline read() {
    // 从头部往尾部传
    tail.read();
    return this;
}

/**
 * 出站方法 write()
 */
@Override
public final ChannelFuture write(Object msg) {
    // 从尾部往头部传
    return tail.write(msg);
}
```

无论是入站事件还是出站事件，想要执行下一个 Handler 必须要调用相应事件的传播方法。事件的传播方法定义在接口 ChannelInboundInvoker 和 ChannelOutboundInvoker 中。

### ChannelInboundInvoker

io.netty.channel.ChannelInboundInvoker

入站事件定义在接口 ChannelInboundInvoker 中。

```java
				/**
				 * 把 channel 注册到 eventloop上
				 */
                ChannelHandlerContext#fireChannelRegistered()}
				/**
				 * 连接成功
				 */
                ChannelHandlerContext#fireChannelActive()}
				/**
				 * 网络读
				 */
                ChannelHandlerContext#fireChannelRead(Object)}
				/**
				 * 网络读结束
				 */
                ChannelHandlerContext#fireChannelReadComplete()}
				/**
				 * 异常捕获
				 */
                ChannelHandlerContext#fireExceptionCaught(Throwable)
                /**
				 * Channel 上收到一个用户定义的 Event
				 */
                ChannelHandlerContext#fireUserEventTriggered(Object)}
                ChannelHandlerContext#fireChannelWritabilityChanged()}
                ChannelHandlerContext#fireChannelInactive()}
				/**
				 * 在 Evenloop 上取消注册某个 Channel
				 */
                ChannelHandlerContext#fireChannelUnregistered()}
```

### ChannelOutboundInvoker

io.netty.channel.ChannelOutboundInvoker

出站事件定义在接口 ChannelOutboundInvoker 中。

```java
			/**
			 * 绑定端口
			 */
            ChannelHandlerContext#bind(SocketAddress, ChannelPromise)
            /**
			 * 连接
			 */
            ChannelHandlerContext#connect(SocketAddress, SocketAddress, ChannelPromise)
            /**
             * 网络写
             */
            ChannelHandlerContext#write(Object, ChannelPromise)
            /**
             * 刷新 channel 内的数据
             */
            ChannelHandlerContext#flush()
            /**
             * 网络读
             */
            ChannelHandlerContext#read()
            /**
             * 关闭连接
             */
            ChannelHandlerContext#disconnect(ChannelPromise)
            /**
             * 关闭 Channel
             */
            ChannelHandlerContext#close(ChannelPromise)
            ChannelHandlerContext#deregister(ChannelPromise)
```

### Pipeline 的截断

事件在 Pipeline 传播过程中需要截断，比如用户自定义了某个 ChannelHandler 负责校验格式，由于数据不符合业务要求，就不应再交给后续的 ChannelHandler 处理，此时就需要截断。

对于入站事件，只要把相应的 channelXXX() 方法注释掉(即不调用)就可以了，后续 ChannelHandler 不会再调用。

```java
static class InboundHandlerB extends ChannelInboundHandlerAdapter{
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("回调：InboundHandlerB");
        // super.channelRead(ctx, msg);
    }
}
```

