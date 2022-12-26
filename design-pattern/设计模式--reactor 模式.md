# 设计模式--reactor 模式



## 说明

1. 本文基于 tomcat 8.5.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 介绍

reactor 模式通常应用于网络 IO 场景，高性能的中间件 redis, netty 都在使用。



## 背景

### 原始的网络 IO 模型

最原始的网络 IO 模型，服务器用一个 while 循环，持续监听端口是否有新的 socket 连接，如果有，那么就调用一个处理函数处理。如果没有则会阻塞直到有新的 socket 连接。
```java
while(true) {
	socket = accept();
	handle(socket);
}
```

这种方式最大问题是无法并发，效率太低。如果当前的请求没有处理完，那么后面的请求只能被阻塞，服务器的吞吐量特别低。改进方式使用多线程，也就是很经典的 connection per thread，每一个连接用一个线程处理。

```java
while(true) {
	socket = accept();
	new thread(socket);
}
```

当然每次处理请求 new 一个线程肯定是不合适的，创建销毁开销高，改进使用线程池:

```java
while(true) {
	socket = accept();
	excutor.execute(socket);
}
```

tomcat 早期版本用过这种实现方式。多线程处理请求确实一定程度上提高了服务器的吞吐量，不同的请求由不同的线程负责处理，之前的请求在处理过程中，不会影响到后续的请求，做到了请求级别的线程隔离。

### 原始网络 IO 模型的缺点

1. 线程的职责划分粒度太大，每一个线程把一次请求交互的事情全部做了，包括连接，读取和返回，限制了吞吐量。
2. 应该把一次连接的操作划分为更细的粒度，这些更细的粒度是更小的线程。虽然这样整个线程池的数目会变多，但是线程职责单一，每个线程效率更高。这样演变出了 reactor 模式。



## reactor 模式

### 初版简易 reactor 模式

1. 在 reactor 模式中，负责处理 IO 事件的是 handler(事件处理函数)，典型的 IO 事件有连接，读取和写入，每一个 handler 使用独立的线程处理一种事件。

2. 一个全局 selector，我们把 channel(IO 事件传输的管道，socket 就是一种 channel)和 selector 关联，这个 selector 会不断在 channel 上调用 accept 函数，检测是否有该类型的事件发生。如果没有，那么 selector 线程会被阻塞，如果有则会调用对应的 handler 来处理。

![img](https://image-hosting.jellyfishmix.com/20221222144649.png)

### 主从 reactor 模式

1. 主 reactor 负责监听 socket 连接，accept 连接后传递给从 reactor 处理，主从 reactor 使用不同的独立线程。
2. 主 reactor 使用独立的线程来执行阻塞的 accept 函数监听 socket 连接，这样从 reactor(包含了 selector)不用去执行阻塞的 accpet 操作了，从 reactor 所在线程没有阻塞的操作。
3. tomcat-connector 连接器模块下三大核心功能之一，网络通信的 IO 多路复用(同步非阻塞)实现 NioEndpoint，用的就是主从 reactor 模式。

![img](https://image-hosting.jellyfishmix.com/20221222161905.jpg)