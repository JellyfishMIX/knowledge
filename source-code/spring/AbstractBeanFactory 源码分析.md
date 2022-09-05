# AbstractBeanFactory 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 写作。



## 概述

由图我们直接的看出，AbstractBeanFactory 继承了 FatoryBeanRegistrySupport 的同时，也实现了 ConfigurableBeanFactory，可想而知 AbstractBeanFactory 的功能有多强大。

![image-20220816182851415](https://image-hosting.jellyfishmix.com/20220816182851.png)

### 作用

- api 里是这样说的，是抽象 BeanFactory 的实现类，同时实现了 ConfigurableBeanFactory 的 SPI，提供了所有的功能
- 可以从我们定义的资源 resource 中获取 bean 的属性。
- 提供了单例 bean 的缓存通过 DefaultSingletonBeanRegistry，同时提供了单例和多例和别名的定义等操作。
- 列举了一部分功能，还有很多功能。



