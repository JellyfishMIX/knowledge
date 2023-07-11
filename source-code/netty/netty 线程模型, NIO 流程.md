# netty 线程模型, NIO 流程



## 说明

1. 本文基于 netty 4.1 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## netty 线程模型

三种线程。worker 线程需要多个。boss 线程和 worker 线程通常一个就够用了，也支持扩容为多个线程。

1. boss 线程：负责实现客户端和服务端的连接，把创建好的连接交给 worker 线程。
2. worker 线程：监听读写事件，通过 NIO Selector 机制，一个 worker 线程可以监听注册在此 Selector 上多个连接(SocketChannel)的读写事件。
3. 自定义业务线程：自定义业务逻辑处理。

## netty NIO 流程

1. 服务端事先对外暴露端口，供客户端连接。boss 线程创建一个 SocketChannel 关注 OP_ACCEPT(接收连接)事件。
2. 客户端向服务端对应的接口发出连接请求。
3. 服务端收到客户端的连接请求后，boss 线程建立连接。创建一个 SocketChannel 实例，这个新的 SocketChannel 实例代表客户端和服务端的连接，用来执行后续读写操作。把这个 SocketChannel 注册到 worker 线程持有的 Selector 上，关注 OP_READ(读)事件。
4. 当客户端向服务端发送数据后，worker 线程持有的 Selector 会监听到读事件，然后 worker 线程从 SocketChannel 中读出数据。把读出的数据传递给业务线程池中处理业务逻辑。
5. 业务逻辑处理完成后，业务线程把响应还给 worker 线程。
6. 最终，worker 线程把响应写入 SocketChannel，即把响应发给客户端。

![Netty 工作流程图.png](https://image-hosting.jellyfishmix.com/20230703173506.png)



## netty 服务端初始化相关参数

![image.png](https://image-hosting.jellyfishmix.com/20230703211908.jpeg)

### BACKLOG

回顾 TCP 三次握手流程:

1. 客户端初始状态为 Closed，这时客户端会给服务端发送 syn 请求来告知服务端有 TCP 连接请求，并把客户端的状态改为 SYN-SENT。
2. 服务端收到客户端发来的 syn 请求后，向客户端发送包含 syn 和 ack 的数据包，syn 表示服务端也要发起连接，ack 是确认已收到客户端发来的 syn 请求，并把服务端的状态改为 SYN-RCVD，服务端会认为这是半连接，并把半连接放入 syn_queue 中。
3. 客户端收到服务端的 ack 确认后，会把自己的状态改为 ESTABLISHED，然后向服务端发送 ack 确认数据包。
4. 服务端收到 ack 确认后，这时服务端任务连接建立成功了，于是把 syn_queue 中对应的半连接拿出来放入 accept_queue 中。java NIO SocketChannel#accept 方法执行后，把连接从 accept_queue 中删除。

BACKLOG 参数影响 accept_queue 里的连接数量，accept_queue 表示的是操作系统已经建立好连接，但还没有被用户程序获取的连接数。

![三次握手与BackLog.png](https://image-hosting.jellyfishmix.com/20230703211923.png)

### KeepAlive 参数

1. 保活参数，当 TCP 的两端长期没有数据交互的时候，有可能对方已经挂了。活着的一端会向另一端尝试发送 KeepAlive 保活请求，如果有 ACK，认为连接正常。如果没有，认为连接断了，并关闭还存活的这一端的 socket，这样能够节省系统的 TCP 连接资源。
2. netty 默认不开启 KeepAlive。原因如下:
   1. 有时候会出现短暂的连接故障，而 KeepAlive 机制可能会把连接误杀。
   2. 如果一个设备有大量的 TCP 连接，但是不活跃的 TCP 连接特别多的话，探活请求会浪费大量的网络流量带宽。

#### TCP 的 KeepAlive 和 HTTP 的 Keep-Alive

1. 两者没有任何关系。
2. HTTP 的 Keep-Alive 设置为 true 后表示一个请求响应过后，HTTP 不会关闭 TCP 连接，这个 TCP 连接可以被后面的请求响应复用。
3. TCP 的 KeepAlive 则是前面说的 TCP 探活机制。