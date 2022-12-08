# AbstractBeanFactory, AbstractAutowireCapableBeanFactory 源码分析



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



## AbstractBeanFactory#doGetBean 方法

org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean

真正获取 bean 实例的全参方法。

### 别名处理

1. 入参 beanName 如果是别名，转换成真实的 beanName。

### 先从三级缓存中获取

1. 从 singletonFactories 中的三级缓存中获取。
   1. 在创建单例 bean 时为了解决循环依赖，spring 创建 bean 的原则是，不等 bean 创建完成，就会将创建 bean 的 ObjectFactory 提早暴露。
   2. 也就是将 ObjectFactory 加入到缓存中，一旦下个 bean 创建时需要依赖上个 bean 则直接使用。
2. 如果三级缓存中获取到了，有可能是 FactoryBean(用于获取 bean 的函数)，此时返回的 bean 来自于 FactoryBean#getObject 返回的实例。
3. 如果三级缓存中没获取到，检查 bean 是否处于创建中。原型模式 Prototype 不解决循环依赖，如果 bean 正处于创建中，说明存在循环依赖，抛出异常。

### 尝试从父容器中获取

1. 如果 beanDefinitionMap 中不包括 beanName，则尝试从父容器中获取 bean。
2. 如果不是仅做类型检查，则是创建 bean，要进行记录。
3. 如果指定的 beanName 是子 bean 的话，会同时合并父类的相关属性。

### 先实例化强制依赖

对于 bean 的 dependsOn 强制依赖，需要实例化。bean 的 dependsOn 强制依赖是显式指定的，不允许出现依赖循环，因此不可以用三级缓存解决，抛异常。

1. DefaultSingletonBeanRegistry 中记录依赖关系。
2. 递归地调用 AbstractBeanFactory#getBean 方法获取依赖 bean。

### 不同 scpoe 下的获取 bean 实例

1. 单例模式获取 bean 实例。
   1. 去 DefaultSingletonBeanRegistry 三级缓存中获取 bean。lambda 函数作为 ObjectFactory。
   2. 如果第一级缓存中未获取到，则调用 ObjectFactory#getObject 方法获取 singleton bean 实例。
2. 原型模式获取 bean 实例。
3. 其他 scope 获取 bean 实例。除了预置的 scope，scope 还支持自定义。

### 类型转换

1. 如果获取到的 bean 实例 type，和所需的 requiredType 不同，将获取的 bean 转换为 requiredType 指定的类型。如果未转换成功，抛出异常。
2. 返回获取到的 bean 实例。

```java
	/**
	 * Return an instance, which may be shared or independent, of the specified bean.
	 *
	 * 真正获取 bean 实例的全参方法
	 *
	 * @param name the name of the bean to retrieve
	 * @param requiredType the required type of the bean to retrieve
	 * @param args arguments to use when creating a bean instance using explicit arguments
	 * (only applied when creating a new instance as opposed to retrieving an existing one)
	 * @param typeCheckOnly whether the instance is obtained for a type check,
	 * not for actual use
	 * @return an instance of the bean
	 * @throws BeansException if the bean could not be created
	 */
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		// 如果传入的是别名 alias，这里会被转换成真实的 beanName
		String beanName = transformedBeanName(name);
		Object bean;

		/*
		 * Eagerly check singleton cache for manually registered singletons.
		 *
		 * 从 singletonFactories 中的三级缓存中获取
		 * 在创建单例 bean 时为了解决循环依赖，spring 创建 bean 的原则是，不等 bean 创建完成，就会将创建 bean 的 ObjectFactory 提早暴露。
		 * 也就是将 ObjectFactory 加入到缓存中，一旦下个 bean 创建时需要依赖上个 bean 则直接使用
		 */
		Object sharedInstance = getSingleton(beanName);
		// 如果三级缓存中获取到了
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 返回 bean 实例。有时 sharedInstance 是 FactoryBean，此时返回的 bean 来自于 FactoryBean#getObject 返回的实例。
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		// 如果三级缓存中没获取到
		else {
			/* Fail if we're already creating this bean instance:
			 * We're assumably within a circular reference.
			 *
			 * 原型模式 Prototype 不解决循环依赖，如果 bean 正处于创建中，说明存在循环依赖，抛出异常。
			 */
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			// 获取父容器
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 如果 beanDefinitionMap 中不包括 beanName，则尝试从父容器中获取 bean
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				// 获取原始的 beanName
				String nameToLookup = originalBeanName(name);
				// 下面是一系列从父容器中获取 bean 的逻辑
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			// 如果不是仅做类型检查，则是创建 bean，要进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				/*
				 * 将存储 xml 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition
				 * 如果指定的 beanName 是子 bean 的话，会同时合并父类的相关属性
				 */
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				/*
				 * Guarantee initialization of beans that the current bean depends on.
				 *
				 * 对于 bean 的 dependsOn 强制依赖，需要实例化
				 */
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						// bean 的 dependsOn 强制依赖是显式指定的，不允许出现依赖循环，因此不可以用三级缓存解决，抛异常。
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// DefaultSingletonBeanRegistry 中记录依赖关系
						registerDependentBean(dep, beanName);
						try {
							// 递归地调用 getBean 方法获取依赖 bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				// 单例模式获取 bean 实例
				if (mbd.isSingleton()) {
					/*
					 * 去 DefaultSingletonBeanRegistry 三级缓存中获取 bean。lambda 函数作为 ObjectFactory。
					 * 如果第一级缓存中未获取到，则调用 ObjectFactory#getObject 方法获取 singleton bean 实例。
					 */
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 调用 createBean 方法创建 bean 实例
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				// 原型模式获取 bean 实例
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				// 其他 scope 获取 bean 实例。除了预置的 scope，scope 还支持自定义。
				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		// 如果获取到的 bean 实例 type，和所需的 requiredType 不同，将获取的 bean 转换为 requiredType 指定的类型。
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				// 将获取的 bean 转换为 requiredType 指定的类型
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				// 未转换成功，抛出异常
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
                // 返回转换后的 bean
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		// 返回获取到的 bean 实例
		return (T) bean;
	}
```



## AbstractAutowireCapableBeanFactory#createBean 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])

根据 BeanDefinition 创建 bean 实例。

1. 为 mbd 解析出对应的 bean clazz。
2. 预处理 bean 需要重写的目标名称方法。
3. bean 初始化前的处理，给 BeanPostProcessors 一个机会来返回目标 bean 的代理。
4. 如果 bean 不为 null，说明 BeanPostProcessor 已经通过回调方法返回了 bean 实例，直接返回此 bean 实例。
5. 调用真正创建 bean 的方法 doCreateBean。
6. 返回创建的 bean 实例。

```java
	/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 *
	 * 根据 BeanDefinition 创建 bean 实例
	 *
	 * @see #doCreateBean
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		/* Make sure bean class is actually resolved at this point, and
		 * clone the bean definition in case of a dynamically resolved Class
		 * which cannot be stored in the shared merged bean definition.
		 *
		 * 为 mbd 解析出对应的 bean clazz
		 */
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			// 预处理 bean 需要重写的目标名称方法。
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			/*
			 * Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			 *
			 * bean 初始化前的处理，给 BeanPostProcessors 一个机会来返回目标 bean 的代理
			 */
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			// 如果 bean 不为 null，说明 BeanPostProcessor 已经通过回调方法返回了 bean 实例，直接返回此 bean 实例。
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 调用真正创建 bean 的方法 doCreateBean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			// 返回创建的 bean 实例
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```



## AbstractBeanDefinition#prepareMethodOverrides 方法

org.springframework.beans.factory.support.AbstractBeanDefinition#prepareMethodOverrides

预处理 bean 需要重写的目标名称方法。

1. 判断此 bean 是否有方法需要重写，这里根据 BeanDefinition 中的 methodOverrides 属性来进行判断，为空则表示没有。

2. 预先做检查，如果目标名称方法没有重载情况，直接设置一个标志位。后续就不需要进行方法参数匹配，直接判断标志位即可。

```java
	/**
	 * Validate and prepare the method overrides defined for this bean.
	 * Checks for existence of a method with the specified name.
	 *
	 * 预处理 bean 需要重写的目标名称方法
	 *
	 * @throws BeanDefinitionValidationException in case of validation failure
	 */
	public void prepareMethodOverrides() throws BeanDefinitionValidationException {
		// Check that lookup methods exist and determine their overloaded status.
		// 判断此 bean 是否有方法需要重写，这里根据 BeanDefinition 中的 methodOverrides 属性来进行判断，为空则表示没有
		if (hasMethodOverrides()) {
			// 预先做检查，如果目标名称方法没有重载情况，直接设置一个标志位。后续就不需要进行方法参数匹配，直接判断标志位即可。
			getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
		}
	}
```



## AbstractBeanDefinition#prepareMethodOverride 方法

org.springframework.beans.factory.support.AbstractBeanDefinition#prepareMethodOverride

1. 如果目标名称方法存在重载情况，则 spring 在方法调用及增强时，需要根据参数列表进行匹配，确定最终调用的方法是哪一个。
2. 这里预先做检查，如果目标名称方法没有重载情况，直接设置一个标志位。后续就不需要进行方法参数匹配，直接判断标志位即可。

```java
	/**
	 * Validate and prepare the given method override.
	 * Checks for existence of a method with the specified name,
	 * marking it as not overloaded if none found.
	 *
	 * 如果目标名称方法存在重载情况，则 spring 在方法调用及增强时，需要根据参数列表进行匹配，确定最终调用的方法是哪一个。
	 * 这里预先做检查，如果目标名称方法没有重载情况，直接设置一个标志位。后续就不需要进行方法参数匹配，直接判断标志位即可。
	 *
	 * @param mo the MethodOverride object to validate
	 * @throws BeanDefinitionValidationException in case of validation failure
	 */
	protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
		// 根据需要重写的方法名称，获取 bean 对应的类中关于该方法的重载有几个
		int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
		if (count == 0) {
			throw new BeanDefinitionValidationException(
					"Invalid method override: no method with name '" + mo.getMethodName() +
					"' on class [" + getBeanClassName() + "]");
		}
		else if (count == 1) {
			// Mark override as not overloaded, to avoid the overhead of arg type checking.
			// 如果目标名称方法没有重载情况，直接设置一个标志位
			mo.setOverloaded(false);
		}
	}
```



## AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation

bean 初始化前的处理。

1. 使用 beanPostProcessor 做 bean 初始化前置处理。
2. 如果 bean 初始化前置处理得到了 bean 实例，则使用 beanPostProcessor 做 bean 初始化后置处理。
3. 返回处理后的 bean。

```java
	/**
	 * Apply before-instantiation post-processors, resolving whether there is a
	 * before-instantiation shortcut for the specified bean.
	 *
	 * bean 初始化前的处理
	 *
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @return the shortcut-determined bean instance, or {@code null} if none
	 */
	@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					// 使用 beanPostProcessor 做 bean 初始化前置处理
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						// 如果 bean 初始化前置处理得到了 bean 实例，则使用 beanPostProcessor 做 bean 初始化后置处理
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		// 返回处理后的 bean
		return bean;
	}
```



## AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation

使用 beanPostProcessor 做 bean 初始化前置处理。

1. 遍历 beanFactory 中的 beanPostProcessor。
2. 判断本次迭代的 beanPostProcessor 是否为 InstantiationAwareBeanPostProcessor 的实例。
3. 供扩展的回调方法，自定义一个实现了 InstantiationAwareBeanPostProcessor 接口的 BeanPostProcessor，重写此方法即可在此触发。

```java
	/**
	 * Apply InstantiationAwareBeanPostProcessors to the specified bean definition
	 * (by class and name), invoking their {@code postProcessBeforeInstantiation} methods.
	 * <p>Any returned object will be used as the bean instead of actually instantiating
	 * the target bean. A {@code null} return value from the post-processor will
	 * result in the target bean being instantiated.
	 *
	 * 使用 beanPostProcessor 做 bean 初始化前置处理
	 *
	 * @param beanClass the class of the bean to be instantiated
	 * @param beanName the name of the bean
	 * @return the bean object to use instead of a default instance of the target bean, or {@code null}
	 * @see InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation
	 */
	@Nullable
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		// 遍历 beanFactory 中的 beanPostProcessor
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			// 判断本次迭代的 beanPostProcessor 是否为 InstantiationAwareBeanPostProcessor 的实例
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				// 供扩展的回调方法，自定义一个实现了 InstantiationAwareBeanPostProcessor 接口的 BeanPostProcessor，重写此方法即可在此触发
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```

额外说明:

1. org.springframework.beans.factory.support.AbstractBeanFactory#addBeanPostProcessor 方法可以给当前 beanFactory 添加 BeanPostProcessor。



## AbstractBeanFactory#resolveBeanClass 方法

org.springframework.beans.factory.support.AbstractBeanFactory#resolveBeanClass

为 mbd 解析出对应的 bean clazz，核心逻辑是调用 AbstractBeanFactory#doResolveBeanClass 方法。

```java
	/**
	 * Resolve the bean class for the specified bean definition,
	 * resolving a bean class name into a Class reference (if necessary)
	 * and storing the resolved Class in the bean definition for further use.
	 *
	 * 为 mbd 解析出对应的 bean clazz
	 *
	 * @param mbd the merged bean definition to determine the class for
	 * @param beanName the name of the bean (for error handling purposes)
	 * @param typesToMatch the types to match in case of internal type matching purposes
	 * (also signals that the returned {@code Class} will never be exposed to application code)
	 * @return the resolved bean class (or {@code null} if none)
	 * @throws CannotLoadBeanClassException if we failed to load the class
	 */
	@Nullable
	protected Class<?> resolveBeanClass(RootBeanDefinition mbd, String beanName, Class<?>... typesToMatch)
			throws CannotLoadBeanClassException {

		try {
			// 如果 mbd 设置了 beanClass，直接返回设置的 clazz
			if (mbd.hasBeanClass()) {
				return mbd.getBeanClass();
			}
			// 如果成功获取到系统的安全管理器
			if (System.getSecurityManager() != null) {
				/*
				 * AccessController.doPrivileged 先不用关注。可以参考 link:
				 * https://www.jianshu.com/p/3fe79e24f8a1, https://www.iteye.com/blog/huangyunbin-1942509
				 *
				 * 获取 mbd 配置的 bean className，加载对应的 clazz，并将加载后的 clazz 缓存在 mbd 中
				 */
				return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>)
						() -> doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());
			}
			else {
				// 获取 mbd 配置的 bean className，加载对应的 clazz，并将加载后的 clazz 缓存在 mbd 中
				return doResolveBeanClass(mbd, typesToMatch);
			}
		}
		catch (PrivilegedActionException pae) {
			ClassNotFoundException ex = (ClassNotFoundException) pae.getException();
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
		}
		catch (ClassNotFoundException ex) {
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
		}
		catch (LinkageError err) {
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), err);
		}
	}
```



## AbstractBeanFactory#doResolveBeanClass 方法

org.springframework.beans.factory.support.AbstractBeanFactory#doResolveBeanClass

获取 mbd 配置的 bean className，加载对应的 clazz，并将加载后的 clazz 缓存在 mbd 中。这里 mbd 指的是 MergedBeanDefinition。

1. 获取该工厂加载 bean 用的类加载器 beanClassLoader。
2. 声明一个动态类加载器 dynamicLoader，默认引用 beanClassLoader。
3. 对入参 typesToMatch 进行处理。
4. 从 mbd 中获取配置的 bean className。
5. 评估 mbd 的 className，如果 className 是可解析表达式，会对其进行解析，然后加载 clazz。
6. 根据 bean 的 className 加载 clazz，并把 clazz 缓存在 mbd 中。

```java
	/**
	 * 获取 mbd 配置的 bean className，加载对应的 clazz，并将加载后的 clazz 缓存在 mbd 中
	 * 这里 mbd 指的是 MergedBeanDefinition
	 *
	 * @param mbd
	 * @param typesToMatch
	 * @return
	 * @throws ClassNotFoundException
	 */
	@Nullable
	private Class<?> doResolveBeanClass(RootBeanDefinition mbd, Class<?>... typesToMatch)
			throws ClassNotFoundException {

		// 获取该工厂加载 bean 用的类加载器 beanClassLoader
		ClassLoader beanClassLoader = getBeanClassLoader();
		// 声明一个动态类加载器 dynamicLoader，默认引用 beanClassLoader
		ClassLoader dynamicLoader = beanClassLoader;
		boolean freshResolve = false;

		// 对入参 typesToMatch 进行处理
		if (!ObjectUtils.isEmpty(typesToMatch)) {
			// When just doing type checks (i.e. not creating an actual instance yet),
			// use the specified temporary class loader (e.g. in a weaving scenario).
			ClassLoader tempClassLoader = getTempClassLoader();
			if (tempClassLoader != null) {
				dynamicLoader = tempClassLoader;
				freshResolve = true;
				if (tempClassLoader instanceof DecoratingClassLoader) {
					DecoratingClassLoader dcl = (DecoratingClassLoader) tempClassLoader;
					for (Class<?> typeToMatch : typesToMatch) {
						dcl.excludeClass(typeToMatch.getName());
					}
				}
			}
		}

		// 从 mbd 中获取配置的 bean className
		String className = mbd.getBeanClassName();
		if (className != null) {
			// 评估 mbd 的 className，如果 className 是可解析表达式，会对其进行解析，然后加载 clazz
			Object evaluated = evaluateBeanDefinitionString(className, mbd);
			if (!className.equals(evaluated)) {
				// A dynamically resolved expression, supported as of 4.2...
				if (evaluated instanceof Class) {
					return (Class<?>) evaluated;
				}
				else if (evaluated instanceof String) {
					className = (String) evaluated;
					freshResolve = true;
				}
				else {
					throw new IllegalStateException("Invalid class name expression result: " + evaluated);
				}
			}
			if (freshResolve) {
				// When resolving against a temporary class loader, exit early in order
				// to avoid storing the resolved Class in the bean definition.
				if (dynamicLoader != null) {
					try {
						return dynamicLoader.loadClass(className);
					}
					catch (ClassNotFoundException ex) {
						if (logger.isTraceEnabled()) {
							logger.trace("Could not load class [" + className + "] from " + dynamicLoader + ": " + ex);
						}
					}
				}
				return ClassUtils.forName(className, dynamicLoader);
			}
		}

		// Resolve regularly, caching the result in the BeanDefinition...
		// 根据 bean 的 className 加载 clazz，并把 clazz 缓存在 mbd 中
		return mbd.resolveBeanClass(beanClassLoader);
	}
```



## AbstractAutowireCapableBeanFactory#doCreateBean 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

真正创建 bean 的方法。

1. 如果 bean 的 scope 是单例模式，从 factoryBean 实例缓存中移除。
2. 创建 bean 实例。注意这里只是把对象在内存中创建出来，还没有填充属性和初始化。
3. 获取 bean 封装实例，获取 bean 实例的封装类型，此时 beanType 确定下来了。
4. 后置处理修改 mbd，且仅允许修改一次。
   1. 留一个扩展点可以修改 mbd。注意这里的扩展点能力不局限于修改 mbd。
   2. 例如 AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition 方法调用的 findAutowiringMetadata 方法，可以寻找此 bean 需要注入的位置，把需要注入的位置做缓存。
5. earlySingletonExposure 标记，用来判断是否出现 bean 循环依赖且需要处理。scope 单例 && 允许循环依赖 && 当前 bean 正在创建中，标记为 true。
   1. earlySingletonExposure 标记为 true 时，调用方法 addSingletonFactory，往 singletonFactories 第三级缓存里放入一个 ObjectFactory 回调函数，当其他的 bean 对该 bean 有依赖时，可以提前获取到早期引用。
   2. getEarlyBeanReference 方法就是获取一个引用，里面调用了 SmartInstantiationAwareBeanPostProcessor 的 getEarlyBeanReference 方法。返回的一般都是 bean 本身，在 aop 介入时是 bean 的代理对象。
6. 调用 populateBean 方法，设置 bean 的属性。
7. 调用 initializeBean 方法，初始化 bean。
8. 处理循环依赖，把 bean 创建结果 exposedObject 设置为早期引用。aop 代理情况令做处理。
9. 调用 registerDisposableBeanIfNecessary 方法注册 bean。
10. 返回 bean 创建结果。

```java
	/**
	 * Actually create the specified bean. Pre-creation processing has already happened
	 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
	 * <p>Differentiates between default bean instantiation, use of a
	 * factory method, and autowiring a constructor.
	 *
	 * 真正创建 bean 的方法
	 *
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a new instance of the bean
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		// 如果 bean 的 scope 是单例模式，从 factoryBean 实例缓存中移除
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		// 创建 bean 实例。注意这里只是把对象在内存中创建出来，还没有填充属性和初始化。
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		// 获取 bean 封装实例
		Object bean = instanceWrapper.getWrappedInstance();
		// 获取 bean 实例的封装类型，此时 beanType 确定下来了。
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		// 后置处理修改 mbd
		synchronized (mbd.postProcessingLock) {
			// 是否后置处理修改过 mbd
			if (!mbd.postProcessed) {
				try {
					/*
					 * 留一个扩展点可以修改 mbd。注意这里的扩展点能力不局限于修改 mbd。
					 * 例如 AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition 方法调用的
					 * findAutowiringMetadata 方法，可以寻找此 bean 需要注入的位置，把需要注入的位置做缓存。
					 */
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				// 后置处理修改标记 postProcessed 置为 true
				mbd.postProcessed = true;
			}
		}

		/*
		 * Eagerly cache singletons to be able to resolve circular references
		 * even when triggered by lifecycle interfaces like BeanFactoryAware.
		 *
		 * earlySingletonExposure 标记，用来判断是否出现 bean 循环依赖且需要处理。scope 单例 && 允许循环依赖 && 当前 bean 正在创建中，标记为 true。
		 * 调用方法 addSingletonFactory，往 singletonFactories 第三级缓存里放入一个 ObjectFactory 回调函数，当其他的 bean 对该 bean 有依赖时，可以提前获取到早期引用。
		 * getEarlyBeanReference 方法就是获取一个引用，里面调用了 SmartInstantiationAwareBeanPostProcessor 的 getEarlyBeanReference 方法。
		 * 返回的一般都是 bean 本身，在 aop 介入时是 bean 的代理对象。
		 */
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		// 如果 earlySingletonExposure 标记为 true，说明出现 bean 循环引用，需要处理。
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 增加一个指定的 singleton ObjectFactory 回调函数，此回调函数用于获取早期的 bean(半成品)。
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		// 初始化 bean 实例
		Object exposedObject = bean;
		try {
			// 设置 bean 的属性
			populateBean(beanName, mbd, instanceWrapper);
			// 初始化 bean
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		// 处理循环依赖
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			// 走进这个 if 说明出现循环依赖，earlySingletonReference 只有在出现循环依赖时才不为空
			if (earlySingletonReference != null) {
				// exposedObject == bean 成立时说明没有出现代理对象(没有 aop 介入)。
				// 把 bean 创建结果 exposedObject 设置为早期引用。
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		// 注册 bean
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		// 返回 bean 创建结果
		return exposedObject;
	}
```



## AbstractAutowireCapableBeanFactory#createBeanInstance 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance

创建 bean 实例。注意这里只是把对象在内存中创建出来，还没有填充属性和初始化。

1. 为 mbd 解析出对应的 bean clazz。
2. 如果 beanClass 不是 public && 不允许访问 non-public 成员属性和方法，抛出异常。
3. mbd 是否有 bean 的 instanceSupplier 回调方法，如果有，通过 instanceSupplier 来创建 bean。用于初始化 bean 的回调函数，一旦指定，这个方法会覆盖构造方法和工厂方法。
4. 如果 mbd 指定了 factoryMethod，则通过 factoryMethod 创建 bean。通过 @Bean 注解方法注入的 bean，或者 xml 配置注入的 BeanDefinition 会存在这个 factoryMethod。注入 bean 的方法就是 factoryMethod。
5. 经过上面的步骤，spring 确定没有其他方式来创建 bean，计划使用 beanClass 本身的构造方法创建 bean。但是 beanClass 的构造方法可能有多个，需要确定使用哪一个。这里的 resolved 和 autowireNecessary 是标记，resolved 表示构造函数是否已经解析完成。autowireNecessary 表示是否需要自动装配。
6. 如果这个 beanClass 的构造方法或工厂方法已经解析过，会保存到 mbd.resolvedConstructorOrFactoryMethod 中。
7. 如果已经解析过 beanClass 的构造函数或工厂方法，则使用解析好的构造函数或工厂工厂方法，不需要再次解析。
   1. 如果 bean 启用自动装配，则使用 autowireConstructor 方法返回的构造方法创建 bean
   2. 如果 bean 未启用自动装配，则使用默认的构造方法创建 bean
8. 走到这一步，说明 bean 是第一次加载，所以没有对创建 bean 使用的构造方法进行缓存(resolved 为 false)。根据 args 参数解析构造方法，并将解析出来的构造方法缓存到 mbd 的 resolvedConstructorOrFactoryMethod 属性中。调用 determineConstructorsFromBeanPostProcessors 方法获取指定 beanClass 的构造方法列表。
9. 如果 mbd 设置了首选构造方法，使用首选构造方法创建 bean。如果 mbd 未设置首选构造方法，使用无参构造方法创建 bean。

```java
	/**
	 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
	 * factory method, constructor autowiring, or simple instantiation.
	 *
	 * 创建 bean 实例。注意这里只是把对象在内存中创建出来，还没有填充属性和初始化。
	 *
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a BeanWrapper for the new instance
	 * @see #obtainFromSupplier
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 * @see #instantiateBean
	 */
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 为 mbd 解析出对应的 bean clazz
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		// beanClass 不是 public && 不允许访问 non-public 成员属性和方法，抛出异常。
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		// mbd 是否有 bean 的 instanceSupplier 回调方法，如果有，通过 instanceSupplier 来创建 bean。
		// instanceSupplier 用于初始化 bean 的回调函数，一旦指定，这个方法会覆盖构造方法和工厂方法。
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		/*
		 * 如果 mbd 指定了 factoryMethod，则通过 factoryMethod 创建 bean。
		 * 通过 @Bean 注解方法注入的 bean，或者 xml 配置注入的 BeanDefinition 会存在这个 factoryMethod。
		 * 注入 bean 的方法就是 factoryMethod。
		 */
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		/*
		 * Shortcut when re-creating the same bean...
		 * 经过上面的步骤，spring 确定没有其他方式来创建 bean，计划使用 beanClass 本身的构造方法创建 bean。
		 * 但是 beanClass 的构造方法可能有多个，需要确定使用哪一个。
		 * 这里的 resolved 和 autowireNecessary 是标记，resolved 表示构造函数是否已经解析完成。autowireNecessary 表示是否需要自动装配。
		 */
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				// 如果这个 beanClass 的构造方法或工厂方法已经解析过，会保存到 mbd.resolvedConstructorOrFactoryMethod 中，这里来判断是否已经解析过。
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		// 如果已经解析过 beanClass 的构造函数或工厂方法，则使用解析好的构造函数或工厂工厂方法，不需要再次解析。
		if (resolved) {
			// 如果 bean 启用自动装配，则使用 autowireConstructor 方法返回的构造方法创建 bean
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				// 如果 bean 未启用自动装配，则使用默认的构造方法创建 bean
				return instantiateBean(beanName, mbd);
			}
		}

		/*
		 * Candidate constructors for autowiring?
		 * 走到这一步，说明 bean 是第一次加载，所以没有对创建 bean 使用的构造方法进行缓存(resolved 为 false)。
		 * 根据 args 参数解析构造方法，并将解析出来的构造方法缓存到 mbd 的 resolvedConstructorOrFactoryMethod 属性中。
		 * 调用 determineConstructorsFromBeanPostProcessors 方法获取指定 beanClass 的构造方法列表。
		 */
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		// 如果 mbd 设置了首选构造方法，使用首选构造方法创建 bean
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		// mbd 未设置首选构造方法，使用无参构造方法创建 bean
		return instantiateBean(beanName, mbd);
	}
```



## AbstractAutowireCapableBeanFactory#populateBean 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean

设置 bean 的属性。

1. 如果 BeanWrapper 为空，但 mdb 有需要注入的属性，抛出异常，无法给一个空实例注入属性。
2. 给 InstantiationAwareBeanPostProcessor 最后一次机会在属性设置前来改变 bean。
   1. 遍历当前 beanFactory 的所有 BeanPostProcessor，InstantiationAwareBeanPostProcessor 的实例可执行 postProcessAfterInstantiation 方法。
   2. 只要有一个 InstantiationAwareBeanPostProcessor 返回 false，都会终止属性填充。
3. 获取 mbd 的 autowireMode 自动装配模式，byName or byType，把所有属性统一存入 propertyValues(pvs) 中。
   1. 这里 autowireMode 不是 @Autowired 注解的含义，指的是 xml 方式 bean 标签中设置的一个 BeanDefinition 属性，启用后会对 beanClass 中所有拥有 setter 方法的成员属性当作 bean 进行注入。

4. 如果当前 beanFactory 拥有 InstantiationAwareBeanPostProcessor，执行相关逻辑。
   1. 成员属性值的后置处理，通过 InstantiationAwareBeanPostProcessor 的 postProcessProperties 方法。（老的 api 是 postProcessPropertyValues）。通过注解注入 bean，就是在这里做的。
   2. 成员属性值的后置处理，新 api。spring 的默认实现是 AutowiredAnnotationBeanPostProcessor.postProcessProperties。
   3. AutowiredAnnotationBeanPostProcessor 中完成了 @Autowired, @Value 等注解的自动注入功能。大概逻辑是，获取被 @Autowired 修饰的属性或者方法，如果是属性，则通过 getBean 获取 bean 并注入。如果是 setter 方法，则获取方法参数后，invoke 方法，给属性赋值。
   4. 把通过 InstantiationAwareBeanPostProcessor 获得的 PropertyValues 赋值给局部变量 pvs。
5. bean 的依赖检查。
6. 调用 applyPropertyValues 方法，将获取的 propertyValue 设置到 bean 中。

```java
	/**
	 * Populate the bean instance in the given BeanWrapper with the property values
	 * from the bean definition.
	 *
	 * 设置 bean 的属性
	 *
	 * @param beanName the name of the bean
	 * @param mbd the bean definition for the bean
	 * @param bw the BeanWrapper with bean instance
	 */
	@SuppressWarnings("deprecation")  // for postProcessPropertyValues
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		// 如果 BeanWrapper 为空，但 mdb 有需要注入的属性，抛出异常，无法给一个空实例注入属性。
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.

		// 给 InstantiationAwareBeanPostProcessor 最后一次机会在属性设置前来改变 bean
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			// 遍历当前 beanFactory 的所有 BeanPostProcessor，InstantiationAwareBeanPostProcessor 的实例可执行 postProcessAfterInstantiation 方法
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					// 只要有一个 InstantiationAwareBeanPostProcessor 返回 false，都会终止属性填充。
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		/*
		 * autowireMode 自动装配模式，byName or byType，把所有属性统一存入到 propertyValues(pvs) 中。
		 * 这里 autowireMode 不是 @Autowired 注解的含义，指的是 xml 方式 bean 标签中设置的一个 BeanDefinition 属性，启用后会对 beanClass 中所有拥有 setter 方法的成员属性当作 bean 进行注入。
		 */
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		// 当前 beanFactory 拥有 InstantiationAwareBeanPostProcessor
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		// 需要依赖检查
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		// 如果当前 beanFactory 拥有 InstantiationAwareBeanPostProcessor
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			// 成员属性值的后置处理，通过 InstantiationAwareBeanPostProcessor 的 postProcessProperties 方法。（老的 api 是 postProcessPropertyValues）
			// 通过注解注入 bean，就是在这里做的
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					/*
					 * 成员属性值的后置处理，新 api。spring 的默认实现是 AutowiredAnnotationBeanPostProcessor.postProcessProperties
					 * AutowiredAnnotationBeanPostProcessor 中完成了 @Autowired, @Value 等注解的自动注入功能。
					 * 大概逻辑是，获取被 @Autowired 修饰的属性或者方法，如果是属性，则通过 getBean 获取 bean 并注入。
					 * 如果是 setter 方法，则获取方法参数后，invoke 方法，给属性赋值。
					 */
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						/*
						 * 成员属性值的后置处理，老 api。spring 的默认实现是 AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues
						 */
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
                    // 把通过 InstantiationAwareBeanPostProcessor 获得的 PropertyValues 赋值给局部变量 pvs
					pvs = pvsToUse;
				}
			}
		}
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			// 依赖检查，对应 depends-on 属性，3.0 已弃用
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		// 将获取的 propertyValue 设置到 bean 中
		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```



## AbstractAutowireCapableBeanFactory#initializeBean 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)

初始化 bean。

1. invokeAwareMethods 方法，对特殊的 bean 进行处理: 实现了 Aware, BeanClassLoaderAware, BeanFactoryAware 等接口的 bean 的处理。
2. 在初始化 bean 之前，调用 BeanPostProcessor#postProcessBeforeInitialization 方法，bean 的生命周期函数。
3. 调用 bean 的 init 方法及此阶段相关生命周期函数。
4. 在初始化 bean 之后，调用 BeanPostProcessor#postProcessAfterInitialization 方法，bean 的生命周期函数。
5. 返回 wrappedBean。

```java
	/**
	 * Initialize the given bean instance, applying factory callbacks
	 * as well as init methods and bean post processors.
	 * <p>Called from {@link #createBean} for traditionally defined beans,
	 * and from {@link #initializeBean} for existing bean instances.
	 *
	 * 初始化 bean
	 *
	 * @param beanName the bean name in the factory (for debugging purposes)
	 * @param bean the new bean instance we may need to initialize
	 * @param mbd the bean definition that the bean was created with
	 * (can also be {@code null}, if given an existing bean instance)
	 * @return the initialized bean instance (potentially wrapped)
	 * @see BeanNameAware
	 * @see BeanClassLoaderAware
	 * @see BeanFactoryAware
	 * @see #applyBeanPostProcessorsBeforeInitialization
	 * @see #invokeInitMethods
	 * @see #applyBeanPostProcessorsAfterInitialization
	 */
	protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		// invokeAwareMethods 方法，对特殊的 bean 进行处理: 实现了 Aware, BeanClassLoaderAware, BeanFactoryAware 等接口的 bean 的处理。
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			// 在初始化 bean 之前，调用 BeanPostProcessor#postProcessBeforeInitialization 方法，bean 的生命周期函数
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			// 调用 bean 的 init 方法及此阶段相关生命周期函数
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			// 在初始化 bean 之后，调用 BeanPostProcessor#postProcessAfterInitialization 方法，bean 的生命周期函数
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		// 返回 wrappedBean
		return wrappedBean;
	}
```



## AbstractAutowireCapableBeanFactory#invokeInitMethods 方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods

调用 bean 的 init 方法及此阶段相关生命周期函数。

1. 调用 bean 生命周期函数 InitializingBean#afterPropertiesSet
2. 调用 BeanDefinition 中指定的 init 方法(initMethodName)

```java
	/**
	 * Give a bean a chance to react now all its properties are set,
	 * and a chance to know about its owning bean factory (this object).
	 * This means checking whether the bean implements InitializingBean or defines
	 * a custom init method, and invoking the necessary callback(s) if it does.
	 *
	 * 调用 bean 的 init 方法及此阶段相关生命周期函数
	 *
	 * @param beanName the bean name in the factory (for debugging purposes)
	 * @param bean the new bean instance we may need to initialize
	 * @param mbd the merged bean definition that the bean was created with
	 * (can also be {@code null}, if given an existing bean instance)
	 * @throws Throwable if thrown by init methods or by the invocation process
	 * @see #invokeCustomInitMethod
	 */
	protected void invokeInitMethods(String beanName, Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			// 调用 bean 生命周期函数 InitializingBean#afterPropertiesSet
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		// 调用 BeanDefinition 中指定的 init 方法(initMethodName)
		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

