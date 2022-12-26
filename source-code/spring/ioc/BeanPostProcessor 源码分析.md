# BeanPostProcessor 源码分析



## BeanPostProcessor

什么是 BeanPostProcessor

BeanPostProcessor 是 spring-ioc 提供的重要的扩展接口，spring 内部的很多功能是通过 BeanPostProcessor 完成的(比如 AnnotationAwareAspectJAutoProxyCreator 的注入)。

BeanPostProcessor 接口提供了两个方法分别在 bean 初始化之前和之后进行调用。

1. postProcessBeforeInitialization 方法，在 bean 初始化前调用，此时 bean 的属性注入已经完成，但还未调用 bean 实例的 init 方法进行初始化。
2. postProcessAfterInitialization 方法，在 bean 初始化后调用，此时 bean 的属性注入已经完成，且已经通过 init 方法初始化完了。

```java
public interface BeanPostProcessor {
	/**
	 * 在 bean 初始化前调用，此时 bean 的属性注入已经完成，但还未调用 bean 的 init 方法进行初始化
	 */
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	/**
	 * 在 bean 初始化后调用，此时 bean 的属性注入已经完成，且已经通过 init 方法初始化完了
	 */
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```



## BeanPostProcessor 的调用时机

1. AbstractAutowireCapableBeanFactory#doCreateBean -> AbstractAutowireCapableBeanFactory#initializeBean 方法中，执行 invokeInitMethods 方法，调用 bean 的 init 方法及此阶段相关生命周期函数来初始化 bean。

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
```



## BeanPostProcessor 的创建(注册)

spring-ioc 机制中，bean 的创建并非仅仅通过反射创建就结束了。当 bean 进行实例化创建时，需要依赖于对应的 BeanDefinition 提供对应的信息。

在创建过程中，BeanDefinition 中不仅包含了 bean 的 class 文件信息，还包含了当前 bean 对应 spring-ioc 中的一些属性，比如 scope 作用域, lazyInit 是否懒加载, alias 别名等信息。

bean 的创建有两个步骤: bean 对应的 BeanDefinition 的创建，bean 实例的创建。BeanPostProcessor 本质也是 bean，也遵循这两个步骤。

1. BeanPostProcessor 的 BeanDefinition 创建时机和普通 bean 没有区别，都是在 spring application 启动时执行 AbstractApplicationContext#refresh 方法时的 BeanFactoryPostProcessor 中完成(确切地说是 ConfigurationClassPostProcessor 中完成的)。
2. 由于 BeanPostProcessor 参与了 bean 的创建过程，所以其创建一定在普通 bean 之前。BeanPostProcessor 的创建是在 spring application 启动时，AbstractApplicationContext#refresh 的地方。
   1. refresh 方法调用了 registerBeanPostProcessors 方法，从 beanFactory 中获取到所有 BeanPostProcessor 类型的beanName，通过 BeanFactory#getBean 方法获取到对应实例，进行排序后注册到 BeanFactory.beanPostProcessors 属性中。当需要使用 BeanPostProcessor 时，直接从 beanPostProcessors 中获取即可。

```java
// BeanFactory

/** BeanPostProcessors to apply. */
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();
```



## BeanPostProcessor 的种类

1. BeanPostProcessor 在 spring 中的子类实现非常多，几十个是有的，例如:

   1. 比如 InstantiationAwareBeanPostProcessorAdapter: 作用在 spring 的 bean 加载过程中。

   2. AnnotationAwareAspectJAutoProxyCreator: bean 创建过程中的属性注入时起作用。

   3. AspectJAwareAdvisorAutoProxyCreator: AspectJ 的 aop 功能也需要 BeanPostProcessor 的特性。