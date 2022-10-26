# HierarchicalBeanFactory 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

向上继承、实现关系：

![image-20220914234320485](https://image-hosting.jellyfishmix.com/20220914234320.png)

向上向下所属层次：

![image-20220914234546002](https://image-hosting.jellyfishmix.com/20220914234546.png)

类签名：

```java
/**
 * Sub-interface implemented by bean factories that can be part
 * of a hierarchy.
 *
 * <p>The corresponding {@code setParentBeanFactory} method for bean
 * factories that allow setting the parent in a configurable
 * fashion can be found in the ConfigurableBeanFactory interface.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 07.07.2003
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#setParentBeanFactory
 */
public interface HierarchicalBeanFactory extends BeanFactory
```



## 概述

HierarchicalBeanFactory 提供父子容器的层次关系查找能力。

至于父容器的设置，需要使用 ConfigurableBeanFactory#setParentBeanFactory

由此可看出，spring 的接口设计，把容器的设置和获取给拆开了。



## 源码分析

```java
/**
 * Sub-interface implemented by bean factories that can be part
 * of a hierarchy.
 *
 * <p>The corresponding {@code setParentBeanFactory} method for bean
 * factories that allow setting the parent in a configurable
 * fashion can be found in the ConfigurableBeanFactory interface.
 *
 * 提供父子容器的层次关系查找能力
 * 至于父容器的设置，需要使用 ConfigurableBeanFactory#setParentBeanFactory
 * 由此可看出，spring 的接口设计，把容器的设置和获取给拆开了
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @since 07.07.2003
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#setParentBeanFactory
 */
public interface HierarchicalBeanFactory extends BeanFactory {

	/**
	 * Return the parent bean factory, or {@code null} if there is none.
	 *
	 * 获取当前容器的父容器
	 */
	@Nullable
	BeanFactory getParentBeanFactory();

	/**
	 * Return whether the local bean factory contains a bean of the given name,
	 * ignoring beans defined in ancestor contexts.
	 * <p>This is an alternative to {@code containsBean}, ignoring a bean
	 * of the given name from an ancestor bean factory.
	 *
	 * 判断当前容器是否包含某个 bean，忽略父类容器中的 bean，只在当前容器中查找。
	 *
	 * @param name the name of the bean to query
	 * @return whether a bean with the given name is defined in the local factory
	 * @see BeanFactory#containsBean
	 */
	boolean containsLocalBean(String name);

}
```

这个接口在继承了 BeanFactory 的基础上，增加了两个方法：

`BeanFactory getParentBeanFactory()` 获取当前容器的父容器。

`boolean containsLocalBean(String name)` 判断当前容器是否包含某个 bean，忽略父类容器中的 bean，只在当前容器中查找。