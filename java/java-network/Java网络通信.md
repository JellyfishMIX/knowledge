#Java网络通信



## 认识Socket

Socket，又称套接字，是在不同的进程间进行网络通讯的一种协议、约定或者说是规范。

对于Socket编程，它更多的时候是基于TCP/UDP等协议做的一层封装或者说抽象，是一套系统所提供的用于进行网络通信相关编程的接口。

 <img src="https://image-hosting.jellyfishmix.com/20200623154202.png" alt="建立Socket连接的基本流程" style="zoom:67%;" />

可以看到本质上，Socket是对TCP连接（当然也有可能是UDP等其他连接）协议，在编程层面上的简化和抽象。



## ServerSocket类

创建一个ServerSocket类，同时在运行该语句的计算机的指定端口处建立一个监听服务，如：

```java
ServerSocket serverSocket =new ServerSocket(600)；
```

这里指定提供监听服务的端口是600，一台计算机可以同时提供多个服务，这些不同的服务之间通过端口号来区别，不同的端口号上提供不同的服务。为了随时监听可能的Client端请求，执行如下的语句：

```java
Socket socket = serverSocket.accept();
```

该语句调用了ServerSocket对象的`accept()`方法，这个方法的执行将使Server端的程序处于等待状态，程序将一直阻塞直到捕捉到一个来自Client端的请求，并返回一个用于与该Client端通信的Socket对象。此后Server程序只要向这个Socket对象读写数据，就可以实现向远端的Client端读写数据。结束监听时，关闭ServerSocket：

```java
serverSocket.close();
```

ServerSocket一般仅用于设置端口号和监听，真正进行通信的是Server端的Socket与Client端的Socket。



## Socket类

当Client端需要从Server端获取信息及其他服务时，应创建一个Socket对象:

```java
Socket socket = new Socket(“IP”，600)；
```

Socket类的构造方法有两个参数，第一个参数是欲连接到的Server端所在计算机的IP地址（请注意，是IP，不是域名），第二个参数是该Server机上提供服务的端口号。

如果需要使用域名表示Server端所在计算机的地址：

```
// 用此句代替IP地址，url为你的域名``InetAddress.getByName(``"url"``);
```

Socket对象建立成功之后，就可以在Client端和Server端之间建立一个连接，并通过这个连接在两个端点之间传递数据。利用Socket类的方法`getInputStream()`和`getOutputStream()`分别获得用于向Socket读写数据的输入／输出流。

当Server端和Client端的通信结束时，可以调用Socket类的`close()`方法关闭连接。



## 例题

### 1. 题目

关于 Socket 通信编程，以下描述正确的是：（ ）

- A 客户端通过new ServerSocket()创建TCP连接对象
- B 客户端通过TCP连接对象调用accept()方法创建通信的Socket对象
- C 客户端通过new Socket()方法创建通信的Socket对象
- D 服务器端通过new ServerSocket()创建通信的Socket对象

#### 解答：

- A 错误
  - Client端通过`new Socket()`创建用于通信的Socket对象。
- B 错误
  - Client端的Socket对象是直接`new`出来的。
- C 正确
- D 错误
  - Server端通过`new ServerSocket()`创建ServerSocket对象，ServerSocket对象的`accept()`方法会产生阻塞，阻塞直到捕捉到一个来自Client端的请求。当服务端捕捉到一个来自Client端的请求时，会创建一个Socket对象，使用此Socket对象与Client端进行通信。



## 引用

[Java中ServerSocket与Socket的区别-原文转载未标出处-CSDN](https://blog.csdn.net/wwwjiahuan/article/details/60881489)

[Socket编程入门（基于Java实现）-Lumin-掘金](https://juejin.im/post/5ad9dd61518825671c0e1d71)

