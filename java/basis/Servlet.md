# Servlet



## servlet是运行在服务端的java程序

类似于Servlet是**Serv**er App**let**（运行在服务端的小程序），是来处理请求的业务逻辑的。

而Tomcat其实是Web服务器和Servlet容器的结合体。

什么是Web服务器？

比如，我当前在杭州，你能否用自己的电脑访问我桌面上的一张图片？恐怕不行。我们太习惯通过URL访问一个网站、下载一部电影了。一个资源，如果没有URL映射，那么外界几乎很难访问。而Web服务器的作用说穿了就是：将某个主机上的资源映射为一个URL供外界访问。

什么是Servlet容器？

Servlet容器，顾名思义里面存放着Servlet对象。我们为什么能通过Web服务器映射的URL访问资源？肯定需要写程序处理请求，主要3个过程：

- 接收请求
- 处理请求
- 响应请求

任何一个应用程序，必然包括这三个步骤。其中接收请求和响应请求是共性功能，且没有差异性。访问淘宝和访问京东，都是接收[http://www.xxx.com/brandNo=1](https://link.zhihu.com/?target=http%3A//www.xxx.com/brandNo%3D1)，响应给浏览器的都是JSON数据。于是，大家就把接收和响应两个步骤抽取成Web服务器：

![img](https://image-hosting.jellyfishmix.com/20200701222036.jpg)

但处理请求的逻辑是不同的。没关系，抽取出来做成Servlet，交给程序员自己编写。

当然，随着后期互联网发展，出现了三层架构，所以一些逻辑就从Servlet抽取出来，分担到Service和Dao。

![img](https://image-hosting.jellyfishmix.com/20200701222056.jpg)

但是Servlet并不擅长往浏览器输出HTML页面，所以出现了JSP。

等Spring家族出现后，Servlet开始退居幕后，取而代之的是方便的SpringMVC。SpringMVC的核心组件DispatcherServlet其实本质就是一个Servlet。但它已经自立门户，在原来HttpServlet的基础上，又封装了一条逻辑。



## servlet的生命周期

主要有三个方法：

- `init() ` 初始化阶段
- `service()` 处理客户端请求阶段
- `destroy()` 终止阶段

### init() 初始化阶段

Servlet容器加载Servlet，加载完成后，Servlet容器会创建一个Servlet实例并调用`init()`方法，`init()`方法只会调用一次。
Servlet容器会在一下几种情况加载Servlet：

- Servlet容器启动时自动装载某些servlet，实现这个需要在web.xml文件中添加`<loadstartup>1</load-on-startup>`。
- 在Servlet容器启动后，客户首次向Servlet发送请求。
- Servlet类文件被更新后，重新装载。

### service() 处理客户端请求阶段

- 每收到一个请求，web服务器会开一个新的线程去处理。

- 对于用户的Servlet请求，Servlet容器会创建一个用于特定请求的ServletRequest和ServletResponse。

- 对于tomcat来说，它会将传递来的参数放入一个HashTable中，这是一个String-->String[]的键值映射。

### destory() 终止阶段

Servlet容器会调用Servlet的`destroy()`方法的情况：

- 当web应用被终止。
- Servlet容器终止运行。
- Servlet容器重新装载Servlet新实例。



## Servlet工作原理

客户发送一个请求。Servlet调用`service()`方法对请求进行响应，`service()`方法会对请求的方法进行匹配，进入相应的逻辑层，完成请求的响应。

但是Servler接口和GenericServlet接口中没有doGet()，doPost()等方法，HttpServlet中定义了这些，但是返回的都是Error信息，所以每次定义Servlet都要重写这些方法。

Sertvlet和GenericServlet是不特定于任何协议的，而HttpServlet是特定于Http协议的，所以HttpServlet中的service()方法中将ServletRequest，ServletResponse强转为HttpRequest，HttpResponse，最后调用自己的`service()`方法去完成响应。



## 配置

Servlet过滤器的配置包括两部分：

- 第一部分是过滤器在Web应用中的定义，由`<filter>`表示，包括`<filter-name>`和`<filter-class>`两个必需的子标签。
- 第二部分是过滤器映射的定义，由`<filter-mapping>`标签表示,可以将一个过滤器映射到一个或者多个Servlet或JSP文件，也可以采用`<url-pattern>`将过滤器映射到任意特征的URL。



## 引用/参考

[servlet的本质是什么，它是如何工作的？ - bravo1988的回答 - 知乎](https://www.zhihu.com/question/21416727/answer/690289895)

[servlet生命周期 - dreamdance - 思否](https://segmentfault.com/a/1190000010725979)

