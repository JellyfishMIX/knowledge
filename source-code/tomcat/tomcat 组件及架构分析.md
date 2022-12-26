# tomcat 组件及架构分析



## tomcat 组件间结构

1. tomcat 的核心是两个组件: connector 和 container(内含 engine, host, context, wrapper)。

2. service 可以对外提供服务，一个 service 包含一个 container 和关联的多个 connector。
3. server 在 service 的外层，控制整个 tomcat 的生命周期，一个 server 包含多个 service。

图 1

![img](https://image-hosting.jellyfishmix.com/20221212144559.png)

图 2

![img](https://image-hosting.jellyfishmix.com/20221212205537.png)

图 3

![img](https://img2018.cnblogs.com/blog/728328/201904/728328-20190419214600281-1893562288.png)



## server

1. server: 表示一个正在 JVM 运行的 tomcat实例(单例)，包含一个或多个 service 子容器。
2. 通过 server.xml 配置文件来配置，其根元素 \<Server\> 代表的正是 tomcat 实例，默认实现为org.apache.catalina.core.StandardServer。也可以通过 \<Server\> 标签的 class 属性指定服务器实现。



## service

service 代表 tomcat 中一组处理请求，提供服务的组件。包括多个 connector 和一个 container。



## connector

connector 链接器封装了底层的网络请求(Socket请求及相应处理),提供了统一的接口，使Container容器与具体的请求协议以及I/O方式解耦。

connector将Socket输入转换成Request对象，交给Container容器进行处理，处理请求后，Container通过Connector提供的Response对象将结果写入输出流。

因为无论是Request对象还是Response对象都没有实现Servlet规范对应的接口，Container会将它们进一步分装成ServletRequest和ServletResponse.



## Engine

Engine容器中，有四个级别的容器，他们的标准实现分别是StandardEngine、StandardHost、StandardContext、StandardWrapper。