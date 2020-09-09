# tomcat与JVM的关系



## 什么是jvm

我们从操作系统的层面来理解，jvm其实就是操作系统中的一个进程。既然是一个进程，那么我们很容易的可以通过任务管理器来查看。假设此时我们启动myeclipse（myeclipse其实就是用java语言编写的一个软件，他的运行必然会启动一个jvm，我们可以把myeclipse理解成我们自己写的一个简单的java版的helloworld程序）。查看任务管理器的截图如下：

![img](https://image-hosting.jellyfishmix.com/20201003171521.png)



## 什么是tomcat

tomcat其实是一个用java语言开发的免费开源的web服务器（因为是java语言开发，这就是为什么使用tomcat前要配置好jdk，因为jdk里面有jvm，而运行java应用需要jvm）。此时再次查看任务管理器会发现多了一个javaw.exe



## tomcat和JVM之间的关系

同一个tomcat下的java ee项目使用的是不是同一个jvm？答案是是的。（使用的都是启动tomcat的jvm）这个可以通过启动不同的web应用来自己判断。

如果运行的是普通的java se程序，使用的是不是同一个jvm呢？答案是否。这个可以自己运行程序判断。（可以写一个很简单的while死循环，便于查看）。



## 引用/参考

[tomcat与jvm的关系分析 - Listen_Silently - CSDN](https://blog.csdn.net/u010653908/article/details/53405395?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.edu_weight&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.edu_weight)