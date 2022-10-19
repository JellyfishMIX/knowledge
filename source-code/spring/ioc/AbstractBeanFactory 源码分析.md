# AbstractBeanFactory 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 写作。
1. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

向上继承、实现关系：

![image-20221018192609408](https://image-hosting.jellyfishmix.com/20221018193312.png)

向上向下所属层次：

![image-20221018193406277](https://image-hosting.jellyfishmix.com/20221018193406.png)

类签名：

```java
public abstract class AbstractBeanFactory extends FactoryBeanRegistrySupport implements ConfigurableBeanFactory
```



## 介绍

1. AbstractBeanFactory 是抽象 BeanFactory 的实现类，实现了 ConfigurableBeanFactory 接口，提供了丰富的功能。
2. 继承了 DefaultSingletonBeanRegistry，提供了 bean 三级缓存机制，提供了别名相关操作。



## AbstractBeanFactory#getBean 方法

org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)

1. 获取 bean 实例，实现了 BeanFactory 接口的 getBean 方法。
2. spring 经常这么做，一个部分入参的方法，封装了内部的全参方法，如 getBean 和 doGetBean 方法。

```java
	/**
	 * 获取 bean 实例，实现了 BeanFactory 接口的 getBean 方法。
	 * spring 经常这么做，一个部分入参的方法，封装了内部的全参方法，如 getBean 和 doGetBean 方法。
	 *
	 * @param name the name of the bean to retrieve
	 * @return
	 * @throws BeansException
	 */
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```



