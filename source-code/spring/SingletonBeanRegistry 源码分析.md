# SingletonBeanRegistry 源码分析



## 前言

1. 本文基于 jdk 8, spring-framework 5.2.x 写作。



## 类层次

向上继承、实现关系：

![image-20220817120814523](https://image-hosting.jellyfishmix.com/20220817120814.png)

向上向下所属层次：

![image-20220817120931481](https://image-hosting.jellyfishmix.com/20220817120931.png)



## 概述

单例(singleton)，是我们使用 Spring IOC 容器时的 bean 实例作用域(scope)。对于单例的维护与管理，Spring 抽象出了 SingletonBeanRegistry 接口来维护。



## 源码分析

```java
public interface SingletonBeanRegistry {

	/**
	 * Register the given existing object as singleton in the bean registry,
	 * under the given bean name.
	 * <p>The given instance is supposed to be fully initialized; the registry
	 * will not perform any initialization callbacks (in particular, it won't
	 * call InitializingBean's {@code afterPropertiesSet} method).
	 * The given instance will not receive any destruction callbacks
	 * (like DisposableBean's {@code destroy} method) either.
	 * <p>When running within a full BeanFactory: <b>Register a bean definition
	 * instead of an existing instance if your bean is supposed to receive
	 * initialization and/or destruction callbacks.</b>
	 * <p>Typically invoked during registry configuration, but can also be used
	 * for runtime registration of singletons. As a consequence, a registry
	 * implementation should synchronize singleton access; it will have to do
	 * this anyway if it supports a BeanFactory's lazy initialization of singletons.
	 *
	 * 单例的注册，如果脱离生命周期单独调用该方法，就是单纯地注册而不执行生命周期回调函数等相关
	 * 通常，是容器在基于 BeanDefinition 创建 singleton bean 时调用该方法的
	 *
	 * @param beanName the name of the bean
	 * @param singletonObject the existing singleton object
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.DisposableBean#destroy
	 * @see org.springframework.beans.factory.support.BeanDefinitionRegistry#registerBeanDefinition
	 */
	void registerSingleton(String beanName, Object singletonObject);

	/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Only checks already instantiated singletons; does not return an Object
	 * for singleton bean definitions which have not been instantiated yet.
	 * <p>The main purpose of this method is to access manually registered singletons
	 * (see {@link #registerSingleton}). Can also be used to access a singleton
	 * defined by a bean definition that already been created, in a raw fashion.
	 * <p><b>NOTE:</b> This lookup method is not aware of FactoryBean prefixes or aliases.
	 * You need to resolve the canonical bean name first before obtaining the singleton instance.
	 *
	 * 获取实例化完成的 singleton bean，可以获取到两种 bean：
	 * 1. 容器基于 BeanDefinition 实例化完成后注册的
	 * 2. 手动 registerSingleton 注册的
	 *
	 * @param beanName the name of the bean to look for
	 * @return the registered singleton object, or {@code null} if none found
	 * @see ConfigurableListableBeanFactory#getBeanDefinition
	 */
	@Nullable
	Object getSingleton(String beanName);

	/**
	 * Check if this registry contains a singleton instance with the given name.
	 * <p>Only checks already instantiated singletons; does not return {@code true}
	 * for singleton bean definitions which have not been instantiated yet.
	 * <p>The main purpose of this method is to check manually registered singletons
	 * (see {@link #registerSingleton}). Can also be used to check whether a
	 * singleton defined by a bean definition has already been created.
	 * <p>To check whether a bean factory contains a bean definition with a given name,
	 * use ListableBeanFactory's {@code containsBeanDefinition}. Calling both
	 * {@code containsBeanDefinition} and {@code containsSingleton} answers
	 * whether a specific bean factory contains a local bean instance with the given name.
	 * <p>Use BeanFactory's {@code containsBean} for general checks whether the
	 * factory knows about a bean with a given name (whether manually registered singleton
	 * instance or created by bean definition), also checking ancestor factories.
	 * <p><b>NOTE:</b> This lookup method is not aware of FactoryBean prefixes or aliases.
	 * You need to resolve the canonical bean name first before checking the singleton status.
	 *
	 * 检查指定 singleton bean 是否存在
	 * 该方法 + ListableBeanFactory#containsBeanDefinition：可以查看当前容器中，是否可以获取到指定 name 的 bean，无论它是否实例化完成
	 * BeanFactory#containsBean 比上面的组合判断(该方法 + ListableBeanFactory#containsBeanDefinition)，多了查找父级容器的逻辑
	 *
	 * @param beanName the name of the bean to look for
	 * @return if this bean factory contains a singleton instance with the given name
	 * @see #registerSingleton
	 * @see org.springframework.beans.factory.ListableBeanFactory#containsBeanDefinition
	 * @see org.springframework.beans.factory.BeanFactory#containsBean
	 */
	boolean containsSingleton(String beanName);

	/**
	 * Return the names of singleton beans registered in this registry.
	 * <p>Only checks already instantiated singletons; does not return names
	 * for singleton bean definitions which have not been instantiated yet.
	 * <p>The main purpose of this method is to check manually registered singletons
	 * (see {@link #registerSingleton}). Can also be used to check which singletons
	 * defined by a bean definition have already been created.
	 *
	 * 获取在此容器中注册的 singleton bean 们的名称，返回一个字符串数组
	 *
	 * @return the list of names as a String array (never {@code null})
	 * @see #registerSingleton
	 * @see org.springframework.beans.factory.support.BeanDefinitionRegistry#getBeanDefinitionNames
	 * @see org.springframework.beans.factory.ListableBeanFactory#getBeanDefinitionNames
	 */
	String[] getSingletonNames();

	/**
	 * Return the number of singleton beans registered in this registry.
	 * <p>Only checks already instantiated singletons; does not count
	 * singleton bean definitions which have not been instantiated yet.
	 * <p>The main purpose of this method is to check manually registered singletons
	 * (see {@link #registerSingleton}). Can also be used to count the number of
	 * singletons defined by a bean definition that have already been created.
	 *
	 * 获取在此容器中注册的 singleton bean 的个数
	 *
	 * @return the number of singleton beans
	 * @see #registerSingleton
	 * @see org.springframework.beans.factory.support.BeanDefinitionRegistry#getBeanDefinitionCount
	 * @see org.springframework.beans.factory.ListableBeanFactory#getBeanDefinitionCount
	 */
	int getSingletonCount();

	/**
	 * Return the singleton mutex used by this registry (for external collaborators).
	 *
	 * 获取这个容器使用的单例锁对象
	 *
	 * @return the mutex object (never {@code null})
	 * @since 4.2
	 */
	Object getSingletonMutex();

}
```

SingletonBeanRegistry 只是一个接口，声明了对于单例维护与管理需要的方法。经典的 AbstractBeanFactory 就是 SingletonBeanRegistry 的一个实现类，可以理解为实现这个接口的实现类，拥有对于单例 bean 的维护与管理的能力。