# AbstractApplicationContext 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## AbstractApplicationContext#refresh 方法

org.springframework.context.support.AbstractApplicationContext#refresh

刷新 spring 应用的上下文。

此方法大量使用了模版方法模式，规定了刷新应用上下文统一的动作，具体动作实现逻辑可供子类定制化。

1. 获取启动关闭锁，线程安全地刷新 spring 应用上下文。
2. 为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。
3. 初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，也就是可以进行 Bean 的提取等基础操作了。
4. 对 beanFactory 做了一些准备工作，进行各种功能填充。
5. postProcessBeanFactory 留给子类扩展，可以对 BeanFactory 做后置处理。
6. 调用各种 BeanFactoryPostProcessor(BeanFactory 处理器)。其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。
7. 注册 BeanPostProcessor。
8. 为上下文初始化 MessageSource，可以对不同语言的消息体进行国际化处理。
9. 初始化应用事件广播器。
10. onRefresh 留给子类的扩展方法，是一个 Context 的生命周期函数。例如子类可以用来初始化其他 bean。
11. 把 ApplicationListener 的 bean 实例添加进应用事件广播器。
12. 非延迟地初始化剩下的实例，这里实例化 bean 调用了 ConfigurableListableBeanFactory#getBean 方法。
13. 完成刷新过程，调用生命周期处理器 LifecycleProcessor#onRefresh 方法。并且发布 ContextRefreshEvent。

```java
	/**
	 * 刷新 spring 应用的上下文
	 * 此方法大量使用了模版方法模式，规定了刷新应用上下文统一的动作，具体动作实现逻辑可供子类定制化。
	 */
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		// 获取启动关闭锁，线程安全地刷新 spring 应用上下文
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。
			prepareRefresh();

			/*
			 * Tell the subclass to refresh the internal bean factory.
			 *
			 * 初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，可以进行 bean 的获取等基础操作了。
			 */
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 对 beanFactory 做了一些准备工作，进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 留给子类扩展，可以对 BeanFactory 做后置处理
				postProcessBeanFactory(beanFactory);

				/*
				 * Invoke factory processors registered as beans in the context.
				 *
				 * 调用各种 BeanFactoryPostProcessor(BeanFactory 处理器)
				 * 其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。
				 */
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册 BeanPostProcessor
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 为上下文初始化 MessageSource，可以对不同语言的消息体进行国际化处理
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化应用事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 留给子类的扩展方法，是一个 Context 的生命周期函数。例如子类可以用来初始化其他 bean
				onRefresh();

				// Check for listener beans and register them.
				// 把 ApplicationListener 的 bean 实例添加进应用事件广播器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 非延迟地初始化剩下的实例，这里实例化 bean 调用了 ConfigurableListableBeanFactory#getBean 方法
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 完成刷新过程，调用生命周期处理器 LifecycleProcessor#onRefresh 方法。并且发布 ContextRefreshEvent。
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```



## AbstractApplicationContext#prepareRefresh 方法

org.springframework.context.support.AbstractApplicationContext#prepareRefresh

为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。

1. 设置启动时间，激活状态。
2. 留给子类实现初始化逻辑的回调函数，可以初始化一些属性资源。此方法 spring 没有默认的实现。
3. 验证需要的属性是否都已经放入环境中。
4. 初始化一些属性。

```java
	/**
	 * Prepare this context for refreshing, setting its startup date and
	 * active flag as well as performing any initialization of property sources.
	 *
	 * 为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。
	 */
	protected void prepareRefresh() {
		// Switch to active.
		// 设置启动时间，激活状态
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		// 留给子类实现初始化逻辑的回调函数，可以初始化一些属性资源。此方法 spring 没有默认的实现。
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		// 验证需要的属性是否都已经放入环境中
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		// 初始化一些属性
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```



## AbstractApplicationContext#obtainFreshBeanFactory 方法

org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，可以进行 bean 的获取等基础操作了。

```java
	/**
	 * Tell the subclass to refresh the internal bean factory.
	 * 
	 * 初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，可以进行 bean 的获取等基础操作了。
	 * 
	 * @return the fresh BeanFactory instance
	 * @see #refreshBeanFactory()
	 * @see #getBeanFactory()
	 */
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```

将 BeanFactory 的创建委托给了 refreshBeanFactory 方法，refreshBeanFactory 方法被两个类实现 AbstractRefreshableApplicationContext 和 GenericApplicationContext。我们这里分析 GenericApplicationContext 的实现。

### GenericApplicationContext#refreshBeanFactory

org.springframework.context.support.GenericApplicationContext#refreshBeanFactory

主要是设置当前 ApplicationContext 的刷新标记，表示当前 ApplicationContext 被刷新了。

```java
	/**
	 * Do nothing: We hold a single internal BeanFactory and rely on callers
	 * to register beans through our public methods (or the BeanFactory's).
	 * @see #registerBeanDefinition
	 */
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		// 设置当前 ApplicationContext 的刷新标记，表示当前 ApplicationContext 被刷新了。
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```

### GenericApplicationContext#getBeanFactory

org.springframework.context.support.GenericApplicationContext#getBeanFactory

this.beanFactory 的实际类型为 DefaultListableBeanFactory，在 GenericApplicationContext 的构造方法中进行了赋值。

```java
	/**
	 * Create a new GenericApplicationContext.
	 * @see #registerBeanDefinition
	 * @see #refresh
	 */
	public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}

	/**
	 * Return the single internal BeanFactory held by this context
	 * (as ConfigurableListableBeanFactory).
	 *
	 * this.beanFactory 的实际类型为 DefaultListableBeanFactory，在 GenericApplicationContext 的构造方法中进行了赋值。
	 */
	@Override
	public final ConfigurableListableBeanFactory getBeanFactory() {
		return this.beanFactory;
	}
```



## AbstractApplicationContext#prepareBeanFactory 方法

org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory

对 beanFactory 做了一些准备工作，进行各种功能填充。

1. 设置当前 beanFactory 的 classLoader 为当前 context 的 classLoader。
2. 给 beanFactory 设置 SpEL 解析器，Spring3 增加了对 SpEL(Spring Expression Language) 的支持。
3. 向 beanFactory 中添加一个属性编辑器。
4. 设置了几个忽略自动装配的接口。
5. 设置了几个自动装配的特殊规则。
6. 向 beanFactory 中增加了一个 BeanPostProcessor: ApplicationListenerDetector。
7. 增加对 AspectJ 的支持。
8. 将相关环境变量及属性的 bean 以单例模式注册。

```java
	/**
	 * Configure the factory's standard context characteristics,
	 * such as the context's ClassLoader and post-processors.
	 *
	 * 对 beanFactory 做了一些准备工作，进行各种功能填充
	 *
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		// 设置当前 beanFactory 的 classLoader 为当前 context 的 classLoader
		beanFactory.setBeanClassLoader(getClassLoader());
		// 给 beanFactory 设置 SpEL 解析器，Spring3 增加了对 SpEL(Spring Expression Language) 的支持。
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        // 向 beanFactory 中添加一个属性编辑器
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		// 设置了几个忽略自动装配的接口
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		// 设置了几个自动装配的特殊规则
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		// 增加对 AspectJ 的支持
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		// 添加默认的系统环境 bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

### SpEL(Spring Expression Language) 的支持

SpEL 使用 #{…} 作为界定符，所有在大括号里面的字符都被认为是SpEL，使用格式如下

```xml
    <bean id="demoA" name="demoA" class="com.kingfish.springbootdemo.replace.DemoA" >
    </bean>
    <bean id="demoB" name="demoB" class="com.kingfish.springbootdemo.replace.DemoB">
        <property name="demoA" value="#{demoA}"/>
    </bean>
```

然后在 AbstractApplicationContext#prepareBeanFactory 方法中的如下代码，可以给 beanFactory 设置 SpEL 解析器，开启对 SpEL 解析的支持。

```java
beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
```

1. SpEL 的解析过程是在 bean 的属性注入阶段(AbstractAutowireCapableBeanFactory#populateBean) 中调用了 applyPropertyValues 方法。
2. applyPropertyValues 方法中通过创建 BeanDefinitionValueResolver 实例 valueResolver 来进行 bean 属性值的解析，在这个步骤中通过 AbstractBeanFactory#evaluateBeanDefinitionString 方法完成了 SpEL 的解析。



## AbstractApplicationContext#postProcessBeanFactory 方法

org.springframework.context.support.AbstractApplicationContext#postProcessBeanFactory

没有默认实现，留给子类扩展，可以对 BeanFactory 做后置处理。

### AbstractRefreshableWebApplicationContext#postProcessBeanFactory 的实现

org.springframework.web.context.support.AbstractRefreshableWebApplicationContext#postProcessBeanFactory

展示一下子类实现扩展方法后可以做的事情，可以给 beanFactory 添加一些 BeanPostProcessor，设置一些 beanFactory 的属性等，可以根据需要对 beanFacotry 做处理。

```java
	/**
	 * Register request/session scopes, a {@link ServletContextAwareProcessor}, etc.
	 */
	@Override
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new ServletContextAwareProcessor(this.servletContext, this.servletConfig));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
		beanFactory.ignoreDependencyInterface(ServletConfigAware.class);

		WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.servletContext, this.servletConfig);
	}
```



## AbstractApplicationContext#invokeBeanFactoryPostProcessors 方法

org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors

调用各种 BeanFactoryPostProcessor(BeanFactory 处理器)

1. BeanFactory 作为 spring-ioc 的基础，用于存放所有已经加载的 bean。为了扩展性，spring 通过 BeanFactoryPostProcessor，支持对 BeanFactory 做处理。
2. 主要是调用各种 BeanFactoryPostProcessors，对当前应用上下文中的 BeanFactory 做处理。其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。

```java
	/**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 *
	 * 调用各种 BeanFactoryPostProcessor(BeanFactory 处理器)
	 * BeanFactory 作为 spring-ioc 的基础，用于存放所有已经加载的 bean。为了扩展性，spring 通过 BeanFactoryPostProcessor，支持对 BeanFactory 做处理。
	 * 主要是调用各种 BeanFactoryPostProcessors，对当前应用上下文中的 BeanFactory 做处理。其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```



## AbstractApplicationContext#registerBeanPostProcessors 方法

org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors

注册 BeanPostProcessor。BeanPostProcessor: 为了扩展性，spring 通过 BeanPostProcessor，支持对 bean 做处理。

1. 获取所有 BeanPostProcessor 的 name(从 beanDefinition 中获得的，没有创建 bean)
2. 给 beanFactory 添加一个 BeanPostProcessorChecker 作为 BeanPostProcessor，BeanPostProcessorChecker 是一个普通的信息打印。有些情况当 spring 中的 BeanPostProcessor 还没有被注册就已经开始了 bean 的实例化，会打印出 BeanPostProcessorChecker 中设定的信息。
3. 创建容纳实现了 PriorityOrdered 接口的 BeanPostProcessorList, 容纳实现了 MergedBeanDefinitionPostProcessor 接口的 BeanPostProcessorList, 容纳实现了 Ordered 接口的 BeanPostProcessorList, 容纳没有实现任何排序接口的 BeanPostProcessorList(常规的 BeanPostProcessor)
4. 按照排序规则分类，把不同的 BeanPostProcessor 放入对应的 list 中。
   1. 第一步，对实现了 PriorityOrdered 接口的 BeanPostProcessor 做排序，然后创建实例并注册进 beanFactory。
   2. 第二步，对实现了 Ordered 接口的 BeanPostProcessor 做排序，然后创建实例并注册进 beanFactory。
   3. 第三步，创建所有常规的 BeanPostProcessor 并注册进 beanFactory。
   4. 第四步(最后一步)，重新注册所有内部的 BeanPostProcessor，即 MergedBeanDefinitionPostProcessor 的实例。这里并不是重复注册， registerBeanPostProcessors 方法会先移除 beanFactory 中已存在的 BeanPostProcessor，然后重新添加进 beanFactory。

5. 给 beanFactory 添加一个用于探测内部 ApplicationListener 实例 bean 的 ApplicationListenerDetector，作为 BeanPostProcessor，把它放在 processorChain 的尾部。

```java
	/**
	 * 注册 BeanPostProcessor，对 BeanPostProcessor 做排序后添加进 beanFactory
	 */
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
		// 获取所有 BeanPostProcessor 的 name(从 beanDefinition 中获得的，没有创建 bean)
		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		// BeanPostProcessorChecker 是一个普通的信息打印
		// 有些情况当 spring 中的 BeanPostProcessor 还没有被注册就已经开始了 bean 的实例化，会打印出 BeanPostProcessorChecker 中设定的信息
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		// 容纳实现了 PriorityOrdered 接口的 BeanPostProcessor
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		// 容纳实现了 MergedBeanDefinitionPostProcessor 接口的 BeanPostProcessor
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		// 容纳实现了 Ordered 接口的 BeanPostProcessor
		List<String> orderedPostProcessorNames = new ArrayList<>();
		// 容纳没有实现任何排序接口的 BeanPostProcessor
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		// 按照排序规则分类，把不同的 BeanPostProcessor 放入对应的 list 中
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		// 第一步，对实现了 PriorityOrdered 接口的 BeanPostProcessor 做排序，然后创建实例并注册进 beanFactory
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		// 把 BeanPostProcessor 注册进 beanFactory，实际上就是保存到 AbstractBeanFactory#beanPostProcessors 集合中，使用 BeanPostProcessor 时从此集合中获取即可。
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		// 第二步，对实现了 Ordered 接口的 BeanPostProcessor 做排序，然后创建实例并注册进 beanFactory
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		// 按照排序规则分类，把不同的 BeanPostProcessor 放入对应的 list 中
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		// 把 BeanPostProcessor 注册进 beanFactory，实际上就是保存到 AbstractBeanFactory#beanPostProcessors 集合中，使用 BeanPostProcessor 时从此集合中获取即可。
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		// 第三步，创建所有常规的 BeanPostProcessor 并注册进 beanFactory
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		// 把 BeanPostProcessor 注册进 beanFactory，实际上就是保存到 AbstractBeanFactory#beanPostProcessors 集合中，使用 BeanPostProcessor 时从此集合中获取即可。
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		// 第四步(最后一步)，重新注册所有内部的 BeanPostProcessor，即 MergedBeanDefinitionPostProcessor 的实例
		sortPostProcessors(internalPostProcessors, beanFactory);
		// 这里并不是重复注册， registerBeanPostProcessors 方法会先移除 beanFactory 中已存在的 BeanPostProcessor，然后重新添加进 beanFactory
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		// 给 beanFactory 添加一个用于探测内部 ApplicationListener 实例 bean 的 ApplicationListenerDetector，作为 BeanPostProcessor，把它放在 processorChain 的尾部
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```



## AbstractApplicationContext#initMessageSource 方法

org.springframework.context.support.AbstractApplicationContext#initMessageSource

为上下文初始化 MessageSource，可以对不同语言的消息体进行国际化处理。获取 name 为 messageSource 的 bean。如果 spring 使用方没有装配这个 bean，spring 提供了默认的 DelegatingMessageSource，将其注册进 beanFactory。

1. 判断 beanFactory 中是否有 name 为 messageSource 的 bean
   1. 如果 beanFactory 中有 messageSource，调用 BeanFactory#getBean 方法来获取 name 为 MESSAGE_SOURCE_BEAN_NAME(messageSource) 的 bean 作为资源文件
   2. 如果 beanFactory 中没有 messageSource，则使用默认的 DelegatingMessageSource 作为 messageSource，以单例模式注册进 beanFactory

```java
	/**
	 * Initialize the MessageSource.
	 * Use parent's if none defined in this context.
	 *
	 * 为上下文初始化 MessageSource，可以对不同语言的消息体进行国际化处理。
	 * 获取 name 为 messageSource 的 bean。
	 * 如果 spring 使用方没有装配这个 bean，spring 提供了默认的 DelegatingMessageSource，将其注册进 beanFactory。
	 */
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		// 判断 beanFactory 中是否有 name 为 messageSource 的 bean
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			// 如果 beanFactory 中有 messageSource，调用 BeanFactory#getBean 方法来获取 name 为 MESSAGE_SOURCE_BEAN_NAME(messageSource) 的 bean 作为资源文件
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			// 如果 beanFactory 中没有 messageSource，则使用默认的 DelegatingMessageSource 作为 messageSource，以单例模式注册进 beanFactory
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
			}
		}
	}
```



## AbstractApplicationContext#initApplicationEventMulticaster 方法

org.springframework.context.support.AbstractApplicationContext#initApplicationEventMulticaster

初始化应用事件广播器。

1. 如果 spring 使用方自行装配了事件广播器，则使用用户自行装配的。
2. spring 使用方没有自行装配，则使用默认的事件广播器 SimpleApplicationEventMulticaster。

```java
	/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 *
	 * 初始化应用事件广播器
	 *
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		// 如果 spring 使用方自行装配了事件广播器，则使用用户自行装配的
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			// spring 使用方没有自行装配，则使用默认的事件广播器 SimpleApplicationEventMulticaster
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

关于应用事件广播器，可以看默认实现 SimpleApplicationEventMulticaster，其负责广播的关键方法是 multicastEvent。

### SimpleApplicationEventMulticaster#multicastEvent 方法

org.springframework.context.event.SimpleApplicationEventMulticaster#multicastEvent(org.springframework.context.ApplicationEvent, org.springframework.core.ResolvableType)

当 spring 的 ApplicationEvent 事件产生的时候，使用 multicastEvent 方法来广播事件。应用了观察者模式。

1. 遍历所有的 ApplicationListener，使用 invokeListener 方法触发每个 listener 的 onApplicationEvent 方法，让 listener 感知到事件的产生。
2. 每个 listener 都可以获取到产生的事件，如何处理由 listener 自行决定。

```java
	/**
	 * 当 spring 的 ApplicationEvent 事件产生的时候，使用 multicastEvent 方法来广播事件。应用了观察者模式。
	 *
	 * @param event the event to multicast
	 * @param eventType the type of event (can be {@code null})
	 */
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		/*
		 * 遍历所有的 ApplicationListener，使用 invokeListener 方法触发每个 listener 的 onApplicationEvent 方法，让 listener 感知到事件的产生
		 * 每个 listener 都可以获取到产生的事件，如何处理由 listener 自行决定
		 */
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```



## AbstractApplicationContext#onRefresh 方法

org.springframework.context.support.AbstractApplicationContext#onRefresh

留给子类的扩展方法，是一个 ApplicationContext 的生命周期函数。例如子类可以用来初始化其他 bean。AbstractApplicationContext 没有对其默认实现。

以子类实现之一 GenericWebApplicationContext#onRefresh 举例，初始化主题资源。

```java
	/**
	 * Initialize the theme capability.
	 */
	@Override
	protected void onRefresh() {
		this.themeSource = UiApplicationContextUtils.initThemeSource(this);
	}
```

额外说一下，springboot 的实现 ServletWebServerApplicationContext#onRefresh，调用了 createWebServer 方法，springboot 在此处会启动 tomcat 服务器。

```java
	protected void onRefresh() {
		super.onRefresh();
		try {
            // springboot 在此处会启动 tomcat 服务器
			createWebServer();
		}
		catch (Throwable ex) {
			throw new ApplicationContextException("Unable to start web server", ex);
		}
	}
```

createWebServer() 方法

```java
	private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			// 获取 webServer 工厂类，因为 webServer 的提供者有多个: JettyServletWebServerFactory, TomcatServletWebServerFactory, UndertowServletWebServerFactory
			ServletWebServerFactory factory = getWebServerFactory();
			// 获取 webServer，其中启动了tomcat
			this.webServer = factory.getWebServer(getSelfInitializer());
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		// 初始化资源
		initPropertySources();
	}
```



## AbstractApplicationContext#registerListeners 方法

org.springframework.context.support.AbstractApplicationContext#registerListeners

把 ApplicationListener 的 bean 实例添加进应用事件广播器。

1. 首先注册硬编码方式的 listener。
2. 从 beanFactory 中获取类型为 ApplicationListener 的 bean 的 beanName。根据 beanName 把 listener 加入应用事件广播器。这里需要是常规的 bean，不能是 FactoryBean。
3. 使用应用事件广播器，发布早期应用程序事件。

```java
	/**
	 * Add beans that implement ApplicationListener as listeners.
	 * Doesn't affect other listeners, which can be added without being beans.
	 *
	 * 把 ApplicationListener 的 bean 实例添加进应用事件广播器
	 */
	protected void registerListeners() {
		// Register statically specified listeners first.
		// 首先注册硬编码方式的 listener
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		/*
		 * Do not initialize FactoryBeans here: We need to leave all regular beans
		 * uninitialized to let post-processors apply to them!
		 *
		 * 从 beanFactory 中获取类型为 ApplicationListener 的 bean 的 beanName。根据 beanName 把 listener 加入应用事件广播器。
		 * 这里需要是常规的 bean，不能是 FactoryBean。
		 */
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		// 使用应用事件广播器，发布早期应用程序事件
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```



## AbstractApplicationContext#finishBeanFactoryInitialization 方法

org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization

1. 设置 conversionService: 如果 beanFactory 中加载了 beanName 为 conversionService 的 bean，并且类型是 ConversionService，那么将其设置为 conversionService。通过 `ConversionService` 的配置可以很轻松完成一些类型转换工作。
2. 冻结 beanFactory 中所有 beanDefinition，表示已注册的 beanDefinition 不会再修改或 post-process
3. 初始化所有非惰性单例 bean。
4. ApplicationContext 默认在启动时，将所有单例 bean 进行实例化。这个实例化的过程就就是在 ConfigurableListableBeanFactory#preInstantiateSingletons 方法中完成的。

```java
	/**
	 * Finish the initialization of this context's bean factory,
	 * initializing all remaining singleton beans.
	 */
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		/*
		 * Initialize conversion service for this context.
		 *
		 * 对 ConversionService 的设置
		 * 如果 beanFactory 中加载了 beanName 为 conversionService 的 bean，并且类型是 ConversionService。那么将其设置为 conversionService
		 */
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no BeanFactoryPostProcessor
		// (such as a PropertySourcesPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		// 调用 getBean 方法初始化 LoadTimeWeaverAware 的实例
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		// 冻结 beanFactory 中所有 beanDefinition，表示已注册的 beanDefinition 不会再修改或 post-process
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 初始化所有非惰性单例 bean，同时考虑 FactoryBean
		beanFactory.preInstantiateSingletons();
	}
```

### 初始化单例 bean 的具体实现 DefaultListableBeanFactory#preInstantiateSingletons 方法

org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons

初始化所有非惰性单例 bean。

1. 获取 beanFactory 中所有 beanDefinition 的 beanName
2. 遍历所有 beanName, 获取合并后的 BeanDefinition
3. 如果是非抽象 && 单例 && 非惰性加载的 bean，则进行初始化。
   1. 如果是 FactoryBean 类型，则拼接 & 这个转译前缀，来获取 FactoryBean 实例本身。
   2. 非 FactoryBean 则直接调用 getBean 来实例化 bean。
4. 触发所有 SmartInitializingSingleton 实例 bean 的 afterSingletonsInstantiated 回调方法，属于生命周期函数。

```java
	/**
	 * 初始化所有非惰性单例 bean
	 */
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		// 获取 beanFactory 中所有 beanDefinition 的 beanName
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		// 遍历所有 beanName, 获取合并后的 BeanDefinition
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// 非抽象 && 单例 && 非惰性加载
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				// 如果是 FactoryBean 类型，则拼接 & 这个转译前缀，来获取 FactoryBean 实例本身
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						// 判断是否要立即初始化 Bean。对于 FactoryBean，可能并不需要立即初始化其 getObject 方法代理的对象。
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				// 非 FactoryBean 则直接调用 getBean 来实例化 bean
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		// 触发所有 SmartInitializingSingleton 实例 bean 的 afterSingletonsInstantiated 回调方法，属于生命周期函数
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

### SmartInitializingSingleton 接口

org.springframework.beans.factory.SmartInitializingSingleton

实现该接口后，当 beanFactory 中所有单例 bean 都初始化完成以后，该接口的方法 afterSingletonsInstantiated 会被触发，可以在此回调函数中做一些事情。属于针对 beanFactory 的生命周期函数。

```java
public interface SmartInitializingSingleton {
	/**
	 * Invoked right at the end of the singleton pre-instantiation phase,
	 * with a guarantee that all regular singleton beans have been created
	 * already. {@link ListableBeanFactory#getBeansOfType} calls within
	 * this method won't trigger accidental side effects during bootstrap.
	 * <p><b>NOTE:</b> This callback won't be triggered for singleton beans
	 * lazily initialized on demand after {@link BeanFactory} bootstrap,
	 * and not for any other bean scope either. Carefully use it for beans
	 * with the intended bootstrap semantics only.
	 */
	void afterSingletonsInstantiated();
}
```



## AbstractApplicationContext#finishRefresh 方法

org.springframework.context.support.AbstractApplicationContext#finishRefresh

完成刷新过程，调用生命周期处理器 LifecycleProcessor#onRefresh 方法，并且发布 ContextRefreshEvent。

1. 清除资源缓存。
2. 当 Application 启动或停止时，会触发 LifecycleProcessor 实现 bean 的回调函数(生命周期函数)。在 LifecycleProcessor 使用前需要初始化，这里进行了 LifecycleProcessor 的初始化。
   1. spring 中还提供的 Lifecycle 接口，包含 start, stop 方法，实现此接口后 spring 保证在 Application 启动时调用其 start 方法(作为生命周期函数)，在 spring 关闭时调用 stop 方法(生命周期函数)，通常用来配置后台程序，在启动后一直运行(如 MQ 进行轮询等)。而 AbstractApplicationContext#finishRefresh 方法保证了这一功能的实现。
3. 当 ApplicationContext#refresh 方法完成的时候，要通过 spring 中的事件发布机制，发布 ContextRefreshedEvent，以保证对应的 listener 可以感知后做进一步的处理。
4. 在 LiveBeansView 中注册 ApplicationContext，LiveBeansView 提供通过 JMX 实时查看 ApplicationContext 里的 bean 列表的能力。

```java
	/**
	 * Finish the refresh of this context, invoking the LifecycleProcessor's
	 * onRefresh() method and publishing the
	 *
	 * 完成刷新过程，调用生命周期处理器 LifecycleProcessor#onRefresh 方法，并且发布 ContextRefreshEvent。
	 *
	 * {@link org.springframework.context.event.ContextRefreshedEvent}.
	 */
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		// 清除资源缓存
		clearResourceCaches();

		/*
		 * Initialize lifecycle processor for this context.
		 *
		 * 当 Application 启动或停止时，会触发 LifecycleProcessor 实现 bean 的回调函数(生命周期函数)
		 * 在 LifecycleProcessor 使用前需要初始化，这里进行了 LifecycleProcessor 的初始化。
		 */
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		// 触发 LifecycleProcessor 实现 bean 的回调函数(生命周期函数)
		getLifecycleProcessor().onRefresh();

		/*
		 * Publish the final event.
		 *
		 * 当 ApplicationContext#refresh 方法完成的时候
		 * 要通过 spring 中的事件发布机制，发布 ContextRefreshedEvent，以保证对应的 listener 可以感知后做进一步的处理
		 */
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		/*
		 * 在 LiveBeansView 中注册 ApplicationContext
		 * LiveBeansView 提供通过 JMX 实时查看 ApplicationContext 里的 bean 列表的能力
		 */
		LiveBeansView.registerApplicationContext(this);
	}
```

