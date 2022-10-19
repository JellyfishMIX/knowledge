# FactoryBeanRegistrySupport 源码分析



## 前言

1. 本文基于 jdk 8, spring-framework 5.2.x 写作。



## 类层次

向上继承、实现关系：

![image-20220917015401240](https://image-hosting.jellyfishmix.com/20220917015401.png)

向上向下所属层次：

![image-20220917015444747](https://image-hosting.jellyfishmix.com/20220917015444.png)

类签名：

```java
/**
 * Support base class for singleton registries which need to handle
 * {@link org.springframework.beans.factory.FactoryBean} instances,
 * integrated with {@link DefaultSingletonBeanRegistry}'s singleton management.
 *
 * FactoryBeanRegistrySupport 在 DefaultSingletonBeanRegistry 基础上，增加了对 FactoryBean 的支持
 *
 * <p>Serves as base class for {@link AbstractBeanFactory}.
 *
 * @author Juergen Hoeller
 * @since 2.5.1
 */
public abstract class FactoryBeanRegistrySupport extends DefaultSingletonBeanRegistry
```



## 概述

FactoryBeanRegistrySupport 在 DefaultSingletonBeanRegistry 基础上，增加了对 FactoryBean 的支持。



## 关键属性

```java
	/**
	 * Cache of singleton objects created by FactoryBeans: FactoryBean name to object.
	 *
	 * 缓存 factoryBean 创建的 singleton 对象
	 */
	private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);
```



## getTypeForFactoryBean 方法

```java
	/**
	 * Determine the type for the given FactoryBean.
	 *
	 * 获取指定 factoryBean 中实例化对象的类型
	 *
	 * @param factoryBean the FactoryBean instance to check
	 * @return the FactoryBean's object type,
	 * or {@code null} if the type cannot be determined yet
	 */
	@Nullable
	protected Class<?> getTypeForFactoryBean(FactoryBean<?> factoryBean) {
		try {
			// 此处是 JDK 的权限控制，主要是操作系统层面的权限。业务开发中很少用到。
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged(
						(PrivilegedAction<Class<?>>) factoryBean::getObjectType, getAccessControlContext());
			}
			else {
				return factoryBean.getObjectType();
			}
		}
		catch (Throwable ex) {
			// Thrown from the FactoryBean's getObjectType implementation.
			logger.info("FactoryBean threw exception from getObjectType, despite the contract saying " +
					"that it should return null if the type of its object cannot be determined yet", ex);
			return null;
		}
	}
```

获取指定 factoryBean 中实例化对象的类型。



## getCachedObjectForFactoryBean 方法

```java
	/**
	 * Obtain an object to expose from the given FactoryBean, if available
	 * in cached form. Quick check for minimal synchronization.
	 *
	 * 从 factoryBeanObjectCache 缓存中，获取指定的 beanName 对象
	 *
	 * @param beanName the name of the bean
	 * @return the object obtained from the FactoryBean,
	 * or {@code null} if not available
	 */
	@Nullable
	protected Object getCachedObjectForFactoryBean(String beanName) {
		return this.factoryBeanObjectCache.get(beanName);
	}
```

从 factoryBeanObjectCache 缓存中，获取指定的 beanName 对象。



## getObjectFromFactoryBean 方法

```java
	/**
	 * Obtain an object to expose from the given FactoryBean.
	 *
	 * 从指定的 FactoryBean 中，获取 beanName 对应的实例对象。
	 * 根据 factoryBean 是否单例，有不同的处理逻辑。并且有前置后置回调函数的预留位置，使得 FactoryBeanRegistrySupport 可以执行子类的重写的回调函数。
	 *
	 * @param factory the FactoryBean instance
	 * @param beanName the name of the bean
	 * @param shouldPostProcess whether the bean is subject to post-processing
	 * @return the object obtained from the FactoryBean
	 * @throws BeanCreationException if FactoryBean object creation failed
	 * @see org.springframework.beans.factory.FactoryBean#getObject()
	 */
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 判断 factoryBean 是否是单例模式，以及 DefaultSingletonBeanFactory 的一级缓存中是否有 beanName 对应的 singleton 实例
		if (factory.isSingleton() && containsSingleton(beanName)) {
			// 针对一级缓存加锁，full singleton lock
			synchronized (getSingletonMutex()) {
				/*
				 * object 对象用于承载返回结果实例对象
				 * 尝试从 factoryBeanObjectCache 缓存中，获取指定的 beanName 对象
				 */
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					// 如果缓存中没有，通过 factoryBean 获取对象
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					// 再次尝试从 factoryBeanObjectCache 缓存中，获取指定的 beanName 对象
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						// factoryBeanObjectCache 缓存中获取到了，赋值给返回结果 object
						object = alreadyThere;
					}
					else {
						// 如果有后置回调方法
						if (shouldPostProcess) {
							// 如果实例正在创建中
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							// 调用 singleton 实例创建的前置回调方法
							beforeSingletonCreation(beanName);
							try {
								// 调用后置回调方法，子类可以重写后置回调方法
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								// 调用 singleton 实例创建的后置回调方法
								afterSingletonCreation(beanName);
							}
						}
						// 如果一级缓存中包含 beanName 对应的 singleton 实例
						if (containsSingleton(beanName)) {
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				// 返回处理后得到的结果实例
				return object;
			}
		}
		else {
			// factoryBean 非单例模式或一级缓存中没有 beanName 对应的 singleton 实例
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			// 如果有后置回调方法
			if (shouldPostProcess) {
				try {
					// 调用后置回调方法，子类可以重写后置回调方法
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			// 返回处理后得到的结果实例
			return object;
		}
	}
```

从指定的 FactoryBean 中，获取 beanName 对应的实例对象。

根据 factoryBean 是否单例，有不同的处理逻辑。并且有前置后置回调函数的预留位置，使得 FactoryBeanRegistrySupport 可以执行子类的重写的回调函数。



## doGetObjectFromFactoryBean 方法

```java
/**
	 * Obtain an object to expose from the given FactoryBean.
	 *
	 * 从指定的 FactoryBean 中，获取 beanName 对应的实例对象。真正在获取实例对象的方法。
	 *
	 * @param factory the FactoryBean instance
	 * @param beanName the name of the bean
	 * @return the object obtained from the FactoryBean
	 * @throws BeanCreationException if FactoryBean object creation failed
	 * @see org.springframework.beans.factory.FactoryBean#getObject()
	 */
	private Object doGetObjectFromFactoryBean(FactoryBean<?> factory, String beanName) throws BeanCreationException {
		// object 对象用于承载返回结果实例对象
		Object object;
		try {
			// 此处是 JDK 的权限控制，主要是操作系统层面的权限。业务开发中很少用到。
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 调用 factoryBean 获得实例
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		if (object == null) {
			// 如果返回结果为 null 且 beanName 对应的实例正在创建中，则抛出异常
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			// 如果返回结果为 null，则创建一个空 Bean(NullBean) 赋值给返回结果
			object = new NullBean();
		}
		// 返回处理后得到的结果实例
		return object;
	}
```

从指定的 FactoryBean 中，获取 beanName 对应的实例对象。真正在获取实例对象的方法。



## postProcessObjectFromFactoryBean 方法

```java
	/**
	 * Post-process the given object that has been obtained from the FactoryBean.
	 * The resulting object will get exposed for bean references.
	 * <p>The default implementation simply returns the given object as-is.
	 * Subclasses may override this, for example, to apply post-processors.
	 *
	 * 后置回调方法，子类可以重写后置回调方法。
	 * 通过这样的回调函数预设，使得 FactoryBeanRegistrySupport 可以执行子类的重写的回调函数。
	 *
	 * @param object the object obtained from the FactoryBean.
	 * @param beanName the name of the bean
	 * @return the object to expose
	 * @throws org.springframework.beans.BeansException if any post-processing failed
	 */
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) throws BeansException {
		return object;
	}
```

后置回调方法，子类可以重写后置回调方法。

通过这样的回调函数预设，使得 FactoryBeanRegistrySupport 可以执行子类的重写的回调函数。



## getFactoryBean 方法

```java
	/**
	 * Get a FactoryBean for the given bean if possible.
	 *
	 * 传入一个普通对象，如果该实例是 factoryBean 的实例，返回一个 factoryBean 实例。否则抛出异常。
	 *
	 * @param beanName the name of the bean
	 * @param beanInstance the corresponding bean instance
	 * @return the bean instance as FactoryBean
	 * @throws BeansException if the given bean cannot be exposed as a FactoryBean
	 */
	protected FactoryBean<?> getFactoryBean(String beanName, Object beanInstance) throws BeansException {
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanCreationException(beanName,
					"Bean instance of type [" + beanInstance.getClass() + "] is not a FactoryBean");
		}
		return (FactoryBean<?>) beanInstance;
	}
```

传入一个普通对象，如果该实例是 factoryBean 的实例，返回一个 factoryBean 实例。否则抛出异常。



## removeSingleton 方法

```java
	/**
	 * Overridden to clear the FactoryBean object cache as well.
	 *
	 * 各级缓存中都根据 beanName 移除对应的 singleton。
	 * FactoryBeanRegistrySupport 除了移除父类 DefaultSingletonBeanRegistry 中的各级缓存，还需要移除 factoryBean 创建的 singleton 对象缓存。
	 */
	@Override
	protected void removeSingleton(String beanName) {
		synchronized (getSingletonMutex()) {
			super.removeSingleton(beanName);
			this.factoryBeanObjectCache.remove(beanName);
		}
	}
```

各级缓存中都根据 beanName 移除对应的 singleton。

FactoryBeanRegistrySupport 除了移除父类 DefaultSingletonBeanRegistry 中的各级缓存，还需要移除 factoryBean 创建的 singleton 对象缓存。



## clearSingletonCache 方法

```java
	/**
	 * Overridden to clear the FactoryBean object cache as well.
	 *
	 * 清空各级缓存中的元素。
	 * FactoryBeanRegistrySupport 除了清空父类 DefaultSingletonBeanRegistry 中的各级缓存，还需要清空 factoryBean 创建的 singleton 对象缓存。
	 */
	@Override
	protected void clearSingletonCache() {
		synchronized (getSingletonMutex()) {
			super.clearSingletonCache();
			this.factoryBeanObjectCache.clear();
		}
	}
```

清空各级缓存中的元素。

FactoryBeanRegistrySupport 除了清空父类 DefaultSingletonBeanRegistry 中的各级缓存，还需要清空 factoryBean 创建的 singleton 对象缓存。

