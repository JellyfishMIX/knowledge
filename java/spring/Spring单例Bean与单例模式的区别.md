# Spring单例Bean与单例模式的区别



## 基本信息

Spring单例Bean与单例模式的区别在于它们关联的环境不一样，单例模式是指在一个JVM进程中仅有一个实例，而Spring单例是指一个Spring Bean容器(ApplicationContext)中仅有一个实例。

首先看单例模式，在一个JVM进程中（理论上，一个运行的JAVA程序就必定有自己一个独立的JVM）仅有一个实例，于是无论在程序中的何处获取实例，始终都返回同一个对象，以Java内置的Runtime为例（现在枚举是单例模式的最佳实践），无论何时何处获取，下面的判断始终为真：

```java
//  基于懒汉模式实现
//  在一个JVM实例中始终只有一个实例
Runtime.getRuntime() == Runtime.getRuntime()
```

与此相比，Spring的单例Bean是与其容器（ApplicationContext）密切相关的，所以在一个JVM进程中，如果有多个Spring容器，即使是单例bean，也一定会创建多个实例，代码示例如下：

```java
//  第一个Spring Bean容器
ApplicationContext context_1 = new FileSystemXmlApplicationContext("classpath:/ApplicationContext.xml");
Person yiifaa_1 = context_1.getBean("yiifaa", Person.class);
//  第二个Spring Bean容器
ApplicationContext context_2 = new FileSystemXmlApplicationContext("classpath:/ApplicationContext.xml");
Person yiifaa_2 = context_2.getBean("yiifaa", Person.class);
//  这里绝对不会相等，因为创建了多个实例
System.out.println(yiifaa_1 == yiifaa_2);
```

以下是Spring的配置文件：

```xml
<!-- 即使声明了为单例，只要有多个容器，也一定会创建多个实例 -->
<bean id="yiifaa" class="com.stixu.anno.Person" scope="singleton">
    <constructor-arg name="username">
        <value>yiifaa</value>
    </constructor-arg>
</bean>
```



## 总结

Spring的单例bean与Spring bean管理容器密切相关，每个容器都会创建自己独有的实例，所以与GOF设计模式中的单例模式相差极大，但在实际应用中，如果将对象的生命周期完全交给Spring管理(不在其他地方通过new、反射等方式创建)，其实也能达到单例模式的效果。



## 引用/参考

[Spring单例Bean与单例模式的区别 - 蚁方阵 - CSDN](https://blog.csdn.net/yiifaa/article/details/74852425)