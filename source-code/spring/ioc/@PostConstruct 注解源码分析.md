# @PostConstruct 注解源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 机制

@PostConstruct 注解加在一个 bean 的方法上，当此 bean 被 spring-ioc 初始化时，自动调用此方法。注意，此类的实例需要作为 bean 交由 spring-ioc 管理时，注解才能生效。

```java
public Object OneBean {
	@PostConstruct
    public void init() {
        // doSomething
    }
}
```

实现原理是借助了 BeanPostProcessor 的机制，具体地说，是 InitDestroyAnnotationBeanPostProcessor。



## BeanPostProcessor 的调用时机

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean

1. spring-ioc 在创建 并初始化 bean 时，AbstractAutowireCapableBeanFactory#doCreateBean -> #initializeBean 方法中，执行 invokeInitMethods 方法，调用 bean 的 init 方法及此阶段相关生命周期函数来初始化 bean。

2. invokeInitMethods 方法执行之前和之后，调用了 BeanPostProcessor 的 postProcessBeforeInitialization 方法，postProcessAfterInitialization 方法。

```java
	/**
	 * 初始化 bean
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

	@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}

	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```



## InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization 方法--实现 BeanPostProcessor 的生命周期方法

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization

实现 BeanPostProcessor 的生命周期方法，在目标 bean 初始化前交由当前 BeanPostProcessor 做处理。

1. 获取 bean 类的生命周期元数据。
2. 调用元数据的初始化方法。

```java
public class InitDestroyAnnotationBeanPostProcessor implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor, PriorityOrdered, Serializable {
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		// 获取 bean 类的生命周期元数据
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			// 调用元数据的初始化方法
			metadata.invokeInitMethods(bean, beanName);
		}
		catch (InvocationTargetException ex) {
			throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
		}
		return bean;
	}
}
```



## InitDestroyAnnotationBeanPostProcessor#findLifecycleMetadata 方法--获取 bean 类的生命周期元数据

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#findLifecycleMetadata

获取 bean 类的生命周期元数据。

1. 生命周期元数据缓存 map，如果为空则创建。
2. 双检锁创建当前 clazz 的生命周期元数据。

```java
	private LifecycleMetadata findLifecycleMetadata(Class<?> clazz) {
		// 生命周期元数据缓存 map 如果为空则创建
		if (this.lifecycleMetadataCache == null) {
			// Happens after deserialization, during destruction...
			return buildLifecycleMetadata(clazz);
		}
		// Quick check on the concurrent map first, with minimal locking.
		// 双检锁创建当前 clazz 的生命周期元数据
		LifecycleMetadata metadata = this.lifecycleMetadataCache.get(clazz);
		if (metadata == null) {
			synchronized (this.lifecycleMetadataCache) {
				metadata = this.lifecycleMetadataCache.get(clazz);
				if (metadata == null) {
					metadata = buildLifecycleMetadata(clazz);
					this.lifecycleMetadataCache.put(clazz, metadata);
				}
				return metadata;
			}
		}
		return metadata;
	}
```



## InitDestroyAnnotationBeanPostProcessor#buildLifecycleMetadata 方法 -- 创建目标 clazz 的生命周期元数据

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor#buildLifecycleMetadata

创建目标 clazz 的生命周期元数据。

1. 声明初始化方法集合和销毁方法集合。
2. 收集目标 clazz 的本地方法(本地方法即不包含从父类继承的)
   1. 收集 init 类型注解标记的方法，至初始化方法集合中。收集 destroy 类型注解标记的方法，至初销毁方法集合中。
   2. 遍历目标 clazz 的父类，重复上述收集过程。

```java
public class InitDestroyAnnotationBeanPostProcessor implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor, PriorityOrdered, Serializable {
	/**
	 * 创建目标 clazz 的生命周期元数据
	 */
	private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
		if (!AnnotationUtils.isCandidateClass(clazz, Arrays.asList(this.initAnnotationType, this.destroyAnnotationType))) {
			return this.emptyLifecycleMetadata;
		}

		// 初始化方法集合
		List<LifecycleElement> initMethods = new ArrayList<>();
		// 销毁方法集合
		List<LifecycleElement> destroyMethods = new ArrayList<>();
		Class<?> targetClass = clazz;

		do {
			// 表示 lambda 中对象不可变，为了代码规整
			final List<LifecycleElement> currInitMethods = new ArrayList<>();
			final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

			// 收集目标 clazz 的本地方法(本地方法即不包含从父类继承的)
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
				// 收集 init 类型注解标记的方法，至初始化方法集合中
				if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
					LifecycleElement element = new LifecycleElement(method);
					currInitMethods.add(element);
					if (logger.isTraceEnabled()) {
						logger.trace("Found init method on class [" + clazz.getName() + "]: " + method);
					}
				}
				// 收集 destroy 类型注解标记的方法，至初销毁方法集合中
				if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
					currDestroyMethods.add(new LifecycleElement(method));
					if (logger.isTraceEnabled()) {
						logger.trace("Found destroy method on class [" + clazz.getName() + "]: " + method);
					}
				}
			});

			initMethods.addAll(0, currInitMethods);
			destroyMethods.addAll(currDestroyMethods);
			targetClass = targetClass.getSuperclass();
			// 遍历目标 clazz 的父类，重复上述收集过程
		} while (targetClass != null && targetClass != Object.class);

		return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
				new LifecycleMetadata(clazz, initMethods, destroyMethods));
	}
}

public abstract class ReflectionUtils {
    /**
     * spring 反射工具类的方法，遍历 clazz 所有本地方法作为入参，触发回调函数。
     */
	public static void doWithLocalMethods(Class<?> clazz, MethodCallback mc) {
		Method[] methods = getDeclaredMethods(clazz, false);
		for (Method method : methods) {
			try {
				mc.doWith(method);
			}
			catch (IllegalAccessException ex) {
				throw new IllegalStateException("Not allowed to access method '" + method.getName() + "': " + ex);
			}
		}
	}
}
```

InitDestroyAnnotationBeanPostProcessor 中使用的 initAnnotationType 和 destroyAnnotationType 成员变量，由子类指定具体值(具体注解)，在子类 CommonAnnotationBeanPostProcessor 中被赋值为 JSR-250 规范提供的 javax.annotation.PostConstruct 和 javax.annotation.PreDestroy。如果使用自定义的 InitDestroyAnnotationBeanPostProcessor 子类，也可以赋值自定义的注解。

```java
public class InitDestroyAnnotationBeanPostProcessor implements DestructionAwareBeanPostProcessor, MergedBeanDefinitionPostProcessor, PriorityOrdered, Serializable {
	@Nullable
	private Class<? extends Annotation> initAnnotationType;

	@Nullable
	private Class<? extends Annotation> destroyAnnotationType;
}

public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {
    public CommonAnnotationBeanPostProcessor() {
		setOrder(Ordered.LOWEST_PRECEDENCE - 3);
		setInitAnnotationType(PostConstruct.class);
		setDestroyAnnotationType(PreDestroy.class);
		ignoreResourceType("javax.xml.ws.WebServiceContext");
	}
}
```



## LifecycleMetadata#invokeInitMethods 方法 -- 调用生命周期元数据的初始化方法

org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor.LifecycleMetadata#invokeInitMethods

遍历 clazz 生命周期元数据的初始化方法并调用。

```java
		/**
		 * 调用生命周期元数据的初始化方法
		 */
		public void invokeInitMethods(Object target, String beanName) throws Throwable {
			Collection<LifecycleElement> checkedInitMethods = this.checkedInitMethods;
			Collection<LifecycleElement> initMethodsToIterate =
					(checkedInitMethods != null ? checkedInitMethods : this.initMethods);
			if (!initMethodsToIterate.isEmpty()) {
				// 遍历 clazz 生命周期元数据的初始化方法
				for (LifecycleElement element : initMethodsToIterate) {
					if (logger.isTraceEnabled()) {
						logger.trace("Invoking init method on bean '" + beanName + "': " + element.getMethod());
					}
					// 调用实际方法
					element.invoke(target);
				}
			}
		}
```

