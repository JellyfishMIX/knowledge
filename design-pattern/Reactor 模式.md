# Reactor 模式



## 什么是 Reactor 模式

wiki：`“The reactor design pattern is an event handling pattern for handling service requests delivered concurrently by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to associated request handlers.”`



## 为什么会有 Reactor 呢

对于应用程序而言，CPU 的处理速度是远远快于 IO 的速度的。如果CPU为了IO操作（例如从Socket读取一段数据）而阻塞显然是不划算的。好一点的方法是分为多进程或者线程去进行处理，但是这样会带来一些进程切换的开销，试想一个进程一个数据读了500ms，期间进程切换到它3次，但是CPU却什么都不能干，就这么切换走了，是不是也不划算？

这时先驱们找到了事件驱动，或者叫回调的方式，来完成这件事情。这种方式就是，应用业务向一个中间人注册一个回调（event handler），当IO就绪后，就这个中间人产生一个事件，并通知此handler进行处理。这种回调的方式，也体现了“好莱坞原则”（Hollywood principle）-“Don’t call us, we’ll call you”，在我们熟悉的IoC中也有用到。看来软件开发真是互通的！



## Reactor 应用场景

Reactor 核心是解决多请求问题。一般来说，Thread-Per-Connection 的应用场景并发量不是特别大，如果并发量过大，会导致线程资源瞬间耗尽，导致服务陷入阻塞，这个时候就需要 Reactor 模式来解决这个问题。Reactor 通过多路复用的思想大大减少线程资源的使用。