# DefaultSingletonBeanRegistry 源码分析



## 前言

1. 本文基于 jdk 8, spring-framework 5.2.x 写作。



## 类层次

向上继承、实现关系：

![image-20220820150703317](https://image-hosting.jellyfishmix.com/20220820150703.png)

向上向下所属层次：

![image-20220820185925782](https://image-hosting.jellyfishmix.com/20220820185925.png)

类签名：

```java
/**
 * Generic registry for shared bean instances, implementing the
 * {@link org.springframework.beans.factory.config.SingletonBeanRegistry}.
 * Allows for registering singleton instances that should be shared
 * for all callers of the registry, to be obtained via bean name.
 *
 * <p>Also supports registration of
 * {@link org.springframework.beans.factory.DisposableBean} instances,
 * (which might or might not correspond to registered singletons),
 * to be destroyed on shutdown of the registry. Dependencies between
 * beans can be registered to enforce an appropriate shutdown order.
 *
 * <p>This class mainly serves as base class for
 * {@link org.springframework.beans.factory.BeanFactory} implementations,
 * factoring out the common management of singleton bean instances. Note that
 * the {@link org.springframework.beans.factory.config.ConfigurableBeanFactory}
 * interface extends the {@link SingletonBeanRegistry} interface.
 *
 * <p>Note that this class assumes neither a bean definition concept
 * nor a specific creation process for bean instances, in contrast to
 * {@link AbstractBeanFactory} and {@link DefaultListableBeanFactory}
 * (which inherit from it). Can alternatively also be used as a nested
 * helper to delegate to.
 *
 * @author Juergen Hoeller
 * @since 2.0
 * @see #registerSingleton
 * @see #registerDisposableBean
 * @see org.springframework.beans.factory.DisposableBean
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory
 */
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
```



## 关键属性

```java
	/**
	 * Cache of singleton objects: bean name to bean instance.
	 *
	 * 单例对象的缓存，key-value: beanName -> bean 实例
	 * 第一级缓存，存放创建完成且初始化完成的 bean
	 * 一级缓存 singletonObjects，在方法中可能会加 synchronized 锁，称为 full singleton lock
	 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/**
	 * Cache of singleton factories: bean name to ObjectFactory.
	 *
	 * 单例工厂的缓存，key-value: beanName -> ObjectFactory
	 * ObjectFactory 可以理解为是用于创建 bean 的回调函数。需要的时候，会调用 ObjectFactory#getObject 去创建 bean
	 * 第三级缓存，存放 bean 的工厂类
	 */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/**
	 * Cache of early singleton objects: bean name to bean instance.
	 *
	 * 早期单例对象的缓存，key-value: beanName -> bean 实例
	 * 第二级缓存，存放半成品 bean。半成品 bean 只是创建了，内存地址分配好了，但是依赖注入还没有做
	 */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```

一级缓存 singletonObjects，在方法中可能会加 synchronized 锁，称为 full singleton lock。DefaultSingletonBeanRegistry 保证线程安全的写法比较简单粗暴，只要想保证线程安全，就给一级缓存加锁，synchronized (this.singletonObjects) {}，即 full singleton lock。

### 三级缓存

这三级 singletonBean 的缓存很重要。简单来说：

1. 一级缓存 singletonObjects 中存放的是创建完成且初始化完成的 bean。
2. 二级缓存 earlySingletonObjects 存放的只是半成品 bean。半成品 bean 只是创建了，内存地址分配好了，但是依赖注入还没有做。
3. 三级缓存 singletonFactories 中存放的是用于创建 bean 的回调函数。需要的时候，会调用 ObjectFactory#getObject 去创建 bean。

三级缓存是为了解决循环依赖问题。

### 其它属性

```java
	/**
	 * Set of registered singletons, containing the bean names in registration order.
	 *
	 * 一组已注册的单例，包含按注册顺序排列的 beanName
	 */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

	/**
	 * Names of beans that are currently in creation.
	 *
	 * 正在创建的单例的 beanName 的集合
	 */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/**
	 * Names of beans currently excluded from in creation checks.
	 *
	 * 创建时不检查的 bean 的集合
	 */
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/**
	 * Collection of suppressed Exceptions, available for associating related causes.
	 *
	 * 异常集合
	 */
	@Nullable
	private Set<Exception> suppressedExceptions;

	/**
	 * Flag that indicates whether we're currently within destroySingletons.
	 *
	 * 当前是否在销毁 bean 中
	 */
	private boolean singletonsCurrentlyInDestruction = false;

	/**
	 * Disposable bean instances: bean name to disposable instance.
	 *
	 * 一次性 bean 实例
	 */
	private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

	/**
	 * Map between containing bean names: bean name to Set of bean names that the bean contains.
	 *
	 * 内部 bean 和外部 bean 之间关系
	 */
	private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

	/**
	 * Map between dependent bean names: bean name to Set of dependent bean names.
	 *
	 * 指定 bean 与依赖指定 bean 的集合，比如 bcd 依赖 a，那么就是 key 为 a，bcd 为 value
	 */
	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

	/**
	 * Map between depending bean names: bean name to Set of bean names for the bean's dependencies.
	 *
	 * 指定 bean 与指定 bean 依赖的集合，比如 a 依赖 bcd，那么就是 key 为 a，bcd 为 value
	 */
	private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```



## registerSingleton 方法

```java
	/**
	 * 把一个对象注册为 bean
	 *
	 * @param beanName the name of the bean
	 * @param singletonObject the existing singleton object
	 * @throws IllegalStateException
	 */
	@Override
	public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
		Assert.notNull(beanName, "Bean name must not be null");
		Assert.notNull(singletonObject, "Singleton object must not be null");
		// synchronized 锁住 singletonObjects，保证对其操作的原子性
		synchronized (this.singletonObjects) {
			Object oldObject = this.singletonObjects.get(beanName);
			// 如果 singletonObjects 不为空，说明之前已经注册过这个 bean 了，按照单例模式的要求，不能再次注册，因此抛出异常。
			if (oldObject != null) {
				throw new IllegalStateException("Could not register object [" + singletonObject +
						"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
			}
			// 单例 bean 加入到缓存中
			addSingleton(beanName, singletonObject);
		}
	}
```

把一个对象注册为单例 bean。真正添加 bean 实例至 map 中调用 addSingleton 方法。



## addSingleton 方法

```java
	/**
	 * Add the given singleton object to the singleton cache of this factory.
	 *
	 * 单例 bean 加入到缓存中
	 *
	 * <p>To be called for eager registration of singletons.
	 * @param beanName the name of the bean
	 * @param singletonObject the singleton object
	 */
	protected void addSingleton(String beanName, Object singletonObject) {
		// synchronized 锁住 singletonObjects，保证对其操作的原子性
		synchronized (this.singletonObjects) {
			// 加入单例对象缓存
			this.singletonObjects.put(beanName, singletonObject);
			// 既然加入了 singletonObjects，说明 bean 对象已经完全创建完毕。singletonFactories 和 earlySingletonObjects 就不再持有。
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			// 加入已经注册的 map
			this.registeredSingletons.add(beanName);
		}
	}
```

单例 bean 加入到一级缓存 singletonObjects 中，同时维护各级缓存。



## addSingletonFactory 方法

```java
	/**
	 * Add the given singleton factory for building the specified singleton
	 * if necessary.
	 *
	 * 增加一个指定的 Singleton ObjectFactory 回调函数
	 *
	 * <p>To be called for eager registration of singletons, e.g. to be able to
	 * resolve circular references.
	 * @param beanName the name of the bean
	 * @param singletonFactory the factory for the singleton object
	 */
	protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		// synchronized 锁住 singletonObjects，保证对其操作的原子性
		synchronized (this.singletonObjects) {
			// 如果 singletonObjects 中已经有 beanName 对应的 bean，说明此 bean 已完全实例化完毕，无需 ObjectFactory
			if (!this.singletonObjects.containsKey(beanName)) {
				// singletonFactories 增加一个回调函数
				this.singletonFactories.put(beanName, singletonFactory);
				// 半实例化缓存中移除 beanName 对应的 bean。此 bean 以后将会调用 ObjectFactory 回调函数进行创建。
				this.earlySingletonObjects.remove(beanName);
				// 已注册的 beanName map 中加入 beanName
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

增加一个指定的 Singleton ObjectFactory 回调函数，同时维护各级缓存。



##  getSingleton(String beanName, boolean allowEarlyReference) 方法

```java
	/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		/*
		* Quick check for existing instance without full singleton lock
		*
		* 快速检查一下一级缓存，看看有没有存在的实例。这样先避免使用 singletonObjects 加锁导致降低效率。
		*/
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			// 快速检查一下二级缓存，看看有没有存在的实例半成品。这样先避免使用 singletonObjects 加锁导致降低效率。
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					// 双检锁中的第二处检查
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						// 双检锁中的第二处检查
						if (singletonObject == null) {
							// 获得用于创建 bean 的回调函数(ObjectFactory)
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								// 使用 singletonFactory#getObject 方法创建 singletonObject
								singletonObject = singletonFactory.getObject();
								// 半成品 bean 放入二级缓存中
								this.earlySingletonObjects.put(beanName, singletonObject);
								// 三级缓存移除回调函数
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}
```

getSingleton 首先快速检查一下一级缓存和二级缓存，看看有没有存在的实例。这样先避免对 singletonObjects 加锁导致降低效率。对于 singletonObjects 和 earlySingletonObjects 都会有双检锁去判断。

如果一级缓存和二级缓存中没有存在的实例，那就看看三级缓存中有没有构造函数。如果三级缓存中有 beanName 对应的构造函数，使用 ObjectFactory#getObject 方法创建 singletonObject。如果一级二级三级缓存中都没有，则会返回 null。 



## getSingleton(String beanName, ObjectFactory<?> singletonFactory) 方法

```java
	/**
	 * Return the (raw) singleton object registered under the given name,
	 * creating and registering a new one if none registered yet.
	 *
	 * 通过 singletonFactory 回调函数获取 singleton bean 实例
	 *
	 * @param beanName the name of the bean
	 * @param singletonFactory the ObjectFactory to lazily create the singleton
	 * with, if necessary
	 * @return the registered singleton object
	 */
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		// 对 singletonObjects 加锁，保证一套操作的原子性，线程安全
		synchronized (this.singletonObjects) {
			// 先尝试去 singletonObjects 中获取 bean
			Object singletonObject = this.singletonObjects.get(beanName);
			// singletonObjects 中未获取到对应的 bean
			if (singletonObject == null) {
				// 当前工厂的 bean 正在销毁中时，不支持获取 bean
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				// singleton 的前置处理。根据 singletonFactory 创建 bean 之前，先检查对应的 bean 是否已经处于创建中
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				// 判断之前是否有被抑制的异常
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				// 把之前被抑制的异常用一个 LinkedHashSet 盛放
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 调用 singletonFactory#getObject 创建 singleton bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime -> 在此期间是否隐式出现单例对象。比如 singletonFactory#getObject 把对象创建后自行放入了 singletonObjects
					// if yes, proceed with it since the exception indicates that state. 如果是，请继续操作，否则继续抛出异常
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					// 记录之前被抑制的异常，和本次异常一起抛出
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					// 重置抑制异常记录，即清除之前记录过的抑制异常
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					// singleton 的创建后置处理
					afterSingletonCreation(beanName);
				}
				// 如果是新增了一个 singleton bean，将单例 bean 加入到缓存中
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

通过 singletonFactory 回调函数获取 singleton bean 实例。创建 singleton bean 实例时，是调用 ObjectFactory#getObject 创建的。



## remove 方法

```java
/**
	 * Remove the bean with the given name from the singleton cache of this factory,
	 * to be able to clean up eager registration of a singleton if creation failed.
	 *
	 * 各级缓存中都根据 beanName 移除对应的 singleton
	 *
	 * @param beanName the name of the bean
	 * @see #getSingletonMutex()
	 */
	protected void removeSingleton(String beanName) {
		// 对一级缓存加锁，然后在各级原子化移除
		synchronized (this.singletonObjects) {
			this.singletonObjects.remove(beanName);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.remove(beanName);
		}
	}
```

各级缓存中都根据 beanName 移除对应的 singleton，这里为了保证各级缓存 remove 的原子性，外面对一级缓存加了一个 synchronized 锁。



## singleton 创建前后回调函数

```java
	/**
	 * Callback before singleton creation.
	 *
	 * singleton 创建的前置处理
	 *
	 * <p>The default implementation register the singleton as currently in creation.
	 * @param beanName the name of the singleton about to be created
	 * @see #isSingletonCurrentlyInCreation
	 */
	protected void beforeSingletonCreation(String beanName) {
		// 根据 singletonFactory 创建 bean 之前，先检查对应的 bean 是否已经处于创建中
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}

	/**
	 * Callback after singleton creation.
	 *
	 * singleton 的创建后置处理
	 *
	 * <p>The default implementation marks the singleton as not in creation anymore.
	 * @param beanName the name of the singleton that has been created
	 * @see #isSingletonCurrentlyInCreation
	 */
	protected void afterSingletonCreation(String beanName) {
		// 根据 singletonFactory 创建 bean 之前，先检查对应的 bean 是否处于创建中
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
		}
	}
```

The default implementation register the singleton as currently in creation. / The default implementation marks the singleton as not in creation anymore.

这两个方法是 singleton bean 创建前后的回调函数。继承 DefaultSingletonBeanRegistry 的子类可以重写这两个方法的逻辑，在 DefaultSingletonBeanRegistry 中创建 singleton bean 时会触发。protected 修饰符意思也是只想让子类能访问到去重写。



## 判断指定的 beanName 是否在创建中

```java
	/**
	 * 判断指定的 beanName 是否在创建中
	 * 作为检测入口，可以做边界条件的处理
	 *
	 * @param beanName
	 * @return
	 */
	public boolean isCurrentlyInCreation(String beanName) {
		Assert.notNull(beanName, "Bean name must not be null");
		// 创建时不检查的 bean 的集合，即白名单。不在白名单中的 beanName，才去检查是否正在创建中。
		return (!this.inCreationCheckExclusions.contains(beanName) && isActuallyInCreation(beanName));
	}

	/**
	 * 判断指定的 beanName 是否实际正在创建中
	 * 对实际的理解：如果有多处需要检查的地方，这个方法可以做一个收口。
	 * 调用这个方法，就能检测到所有需要检测的位置。可以参考子类重写的：org.springframework.beans.factory.support.abstractbeanfactory#isActuallyInCreation(java.lang.String)
	 *
	 * @param beanName
	 * @return
	 */
	protected boolean isActuallyInCreation(String beanName) {
		return isSingletonCurrentlyInCreation(beanName);
	}

	/**
	 * Return whether the specified singleton bean is currently in creation
	 * (within the entire factory).
	 *
	 * 判断指定的 singleton beanName 是否正在创建中
	 *
	 * @param beanName the name of the bean
	 */
	public boolean isSingletonCurrentlyInCreation(String beanName) {
		// singletonsCurrentlyInCreation 中维护了正在创建的单例的 beanName 的集合
		return this.singletonsCurrentlyInCreation.contains(beanName);
	}
```

以上三个方法连用：

1. isCurrentlyInCreation 作为检测入口，可以做边界条件的处理。
2. isActuallyInCreation 判断指定的 beanName 是否实际正在创建中。对实际的理解：如果有多处需要检查的地方，这个方法可以做一个收口。调用这个方法，就能检测到所有需要检测的位置。可以参考子类重写的: org.springframework.beans.factory.support.abstractbeanfactory#isActuallyInCreation(java.lang.String)

```
	/**
	 * AbstractBeanFactory 作为 DefaultSingletonBeanRegistry 子类重写的方法，这里就能看出收口方法的作用了。
	 * 如果有多处需要检查的地方，这个方法可以做一个收口。调用这个方法，就能检测到所有需要检测的位置。
	 */
	@Override
	public boolean isActuallyInCreation(String beanName) {
		return (isSingletonCurrentlyInCreation(beanName) || isPrototypeCurrentlyInCreation(beanName));
	}
```

3. isSingletonCurrentlyInCreation 判断指定的 singleton beanName 是否正在创建中，singletonsCurrentlyInCreation 中维护了正在创建的单例的 beanName 的集合。



## registerDependentBean 方法

```java
	/**
	 * Register a dependent bean for the given bean,
	 * to be destroyed before the given bean is destroyed.
	 *
	 * 把 bean 注册为 dependentBean 的依赖
	 *
	 * @param beanName the name of the bean
	 * @param dependentBeanName the name of the dependent bean
	 */
	public void registerDependentBean(String beanName, String dependentBeanName) {
		// 传入一个 name，不管这个 name 是别名还是本名，此方法将获得真实的本名
		String canonicalName = canonicalName(beanName);

		/*
		 * 指定 bean 与依赖指定 bean 的集合，比如 bcd 依赖 a，那么就是 key 为 a，bcd 为 value
		 * dependentBeanMap 是 ConcurrentHashMap。这里体现了 ConcurrentHashMap 只能保证 api 是线程安全的，如果组合使用多个 api，还是可能出现竟态条件问题，需要加锁
		 */
		synchronized (this.dependentBeanMap) {
			// 拿到 beanName 对应的 value，如果 value 不存在就初始化一个 LinkedHashSet 用于存储依赖于 beanName bean 的 dependentBean
			Set<String> dependentBeans =
					this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
			// beanName bean 的 dependentBean set 中添加一个元素
			if (!dependentBeans.add(dependentBeanName)) {
				return;
			}
		}

		// 指定 bean 与指定 bean 依赖的集合，比如 a 依赖 bcd，那么就是 key 为 a，bcd 为 value
		synchronized (this.dependenciesForBeanMap) {
			// 拿到 dependentBeanName 对应的 value，如果 value 不存在就初始化一个 LinkedHashSet 用于存储 dependentBeanName bean 的 dependenciesBean
			Set<String> dependenciesForBean =
					this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
			// dependentBeanName bean 的 dependenciesBean set 中添加一个元素
			dependenciesForBean.add(canonicalName);
		}
	}
```

把 bean 注册为 dependentBean 的依赖，同时维护了：

1. dependentBeanMap: 指定 bean 与依赖指定 bean 的集合，比如 bcd 依赖 a，那么就是 key 为 a，bcd 为 value。
2. dependenciesForBeanMap: 指定 bean 与指定 bean 依赖的集合，比如 a 依赖 bcd，那么就是 key 为 a，bcd 为 value。

并且此方法中针对 ConcurrentHashMap 加了 synchronized 锁。这里体现了 ConcurrentHashMap 只能保证 api 是线程安全的，如果组合使用多个 api，还是可能出现竟态条件问题，需要加锁。
