# Spring Basis

### 

## Spring的IOC构成

配置文件 + 工厂 + 反射



## Spring工厂类

<img src="https://image-hosting.jellyfishmix.com/20200616122321.png" alt="截屏2020-06-16下午12.23.05" style="zoom:40%;" />

### ClassPathXmlApplicationContext

使用本Project的 `applicationContext.xml`配置文件创建工厂类

### FileSystemXmlApplicationContext

使用指定的文件目录的 `applicationContext.xml` 配置文件创建工厂类

e.g.

`applicationContext.xml`

```xml
<!-- UserServiceImplA的创建权转交给了Spring -->
<bean id="userSerivce" class="com.example.ioc.demo.UserServiceImplA"></bean>
```



`UserServiceImplA.java`

```java
package com.example.ioc.demo.UserServiceImplA

public class UserServiceImplA {
  public void demo() {
    // 创建Spring的工厂
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
    // 通过工厂获得类
    UserService userService = (UserService) applicationContext.getBean("userService");
	}
}
```



### BeanFactory和ApplicationContext

BeanFactory是老的工厂类，ApplicationContext是新的工厂类

主要区别：

- ApplicationContext的功能比BeanFactory的功能多

  比如国际化等功能，ApplicationContext中有，BeanFactory中无

- 生成Bean实例的时机不同

  BeanFactory在工厂实例化后，使用.getBean("")调用时，才把Bean对应类的实例创建出来

  ApplicationContext在加载`applicationContext.xml`配置文件时，就把所有单例模式的Bean对应的实例创建出来


