# linux I/O



## linux中的IO

linux的IO将所有外部设备都看作文件来操作，与外部设备的操作都可以看做文件操作，其读写都使用内核提供的系统调用，内核会返回一个文件描述符（fd, file descriptor），例如socket读写使用socketfd。描述符是一个索引，指向内核中一个结构体，应用程序对文件的读写通过描述符完成。

一个基本的IO，涉及两个系统对象：

- 调用这个IO进程的对象

- 系统内核

read操作发生时流程如下：

1. 通过read系统调用向内核发起读请求。
2. 内核向硬件发送读指令，并等待读就绪。
3. 内核把将要读取的数据复制到描述符所指向的内核缓存区中。
4. 将数据从内核缓存区拷贝到用户进程空间中。



## Linux I/O模型简介

1. 阻塞I/O模型：最常用，所有文件操作都是阻塞的。
2. 非阻塞I/O模型：缓冲区无数据则返回，一般采用轮询的方式做状态检查。
3. I/O复用模型：详细见下文。
4. 信号驱动I/O：使用信号回调应用，内核通知用户何时开启一个I/O操作。
5. 异步I/O：内核操作完成后进行通知，内核通知用户何时完成一个I/O操作。



## Linux IO 多路复用

### 使用场景

- 客户处理多个描述符（交互输入，网络Socket）。
- 客户处理多个Socket（少见）。
- TCP服务器既要处理监听Socket，又要处理已连接Socket。
- 一个服务器既要处理TCP，又要处理UDP。
- 一个服务器处理多个服务/多个协议。

### 与多进程/多线程对比

I/O多路复用系统开销小，系统不必创建进程/线程，也不需要维护这些进程/线程。

### 系统调用

目前支持I/O多路复用的系统调用包括select, pselect, poll, epoll，I/O多路复用即通过一种机制，一个进程可以监视多个描述符，一旦某个描述符准备就绪，就能够通知程序进行相应的读写操作。

### select/poll

select目前在所有平台支持，select函数监视文件操作符（将fd加入fdset），循环遍历fdset内的fd获取是否有资源的信息，若遍历完所有fdset内的fd后无资源可用，则select让该进程睡眠，直到有资源可用或超时则唤醒select进程，之后select继续循环遍历，找到就绪的fd后返回。select单个进程打开的fd有一定限制，由FD_SETSIZE设置，默认为1024（32位）和2048（64位）。

poll与select的主要区别是不使用fdset，而是使用pollfd结构（本质链表结构），因而没有fd数目限制。

poll和select共有的问题：

- 每次select/poll找到就绪的fd，都需要把fdset/pollfd进行内存复制。
- select/poll，都要在内核中遍历所有传递来的fd来寻找就绪的fd，随着监视的fd数量增加，效率也会下降。

### epoll

Linux 2.6内核中提出了epoll，epoll包括epoll_create,epoll_ctl,epoll_wait三个函数分别负责创建epoll，注册监听的事件和等待事件产生。

- epoll每次注册新的事件到epoll中时，都会把所有fd拷贝进内核，而不是在epoll_wait时重复拷贝，保证每个fd在整个过程中仅拷贝一次。此外，epoll将内核与用户进程空间mmap到同一块内存，将fd消息存于该内存避免了不必要的拷贝。
- epoll使用事件的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就通过回调函数把就绪的fd加入一个就绪链表，唤醒epoll_wait进入睡眠的进程，epoll_wait通知消息给应用程序后再次睡眠。因此epoll不随着fd数目增加效率下降，只有活跃fd才会调用回调函数，效率与连接总数无关。
- epoll没有最大并发连接的限制，1G内存约能监听10万个端口。

epoll有LT模式和ET模式：

- LT模式：epoll_wait检测到fd并通知后，应用程序可以不立刻处理，下次调用epoll_wait，会再次通知；
- ET模式：应用程序必须立刻处理，下次调用，不会再通知此事件。ET模式效率更高，epoll工作在ET模式下必须使用非阻塞套接字。

### 性能对比

- 如果有大量的idle-connection或dead-connection，epoll效率比select/poll高很多。
- 连接少连接十分活跃的情况下，select/poll的性能可能比epoll好。

### Java的IO模式

1. BIO：即传统Socket编程，线程数：客户端访问数为1：1，由于线程数膨胀后系统性能会急剧下降，导致了BIO的低效。
2. 伪异步I/O：为了解决一个链路一个线程的问题，引入线程池处理多个客户端接入请求，可以灵活调配线程资源，可以限制线程数量防止膨胀，但底层仍是阻塞模型，高客户端访问时，会有通信阻塞的问题。
3. NIO：Java NIO的核心为Channels, Buffers, Selectors。Channel有点像流，数据可以从Channel读到Buffer中，也可以从Buffer写到Channel内。而Selector则被用于多路复用，Java NIO可以把Channel注册到Selector上，之后，Selector会获取进入就绪状态的Channel（Selector进行循环的select/poll/epoll/IOCP操作），并进行后续操作。Selector是NIO实现的关键。Java NIO编程较为复杂。
4. AIO：NIO.2引入的异步通道概念，不需要多路复用器对注册的通道轮询即可异步读写，简化了NIO的编程。（但是Netty作者称AIO的性能并不比NIO和epoll好）。



## 引用/参考

[Java多线程：Linux多路复用，Java NIO与Netty简述 - CieloSun的博客](https://www.cnblogs.com/cielosun/p/10614351.html)