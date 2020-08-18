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



## 引用/参考

[Java多线程：Linux多路复用，Java NIO与Netty简述 - CieloSun的博客](https://www.cnblogs.com/cielosun/p/10614351.html)