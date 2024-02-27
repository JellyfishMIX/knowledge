# spring-aop 收集切面,创建代理



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 用于创建代理的 BeanPostProcessor -- AnnotationAwareAspectJAutoProxyCreator

1. 实现了 InstantiationAwareBeanPostProcessor 接口，其他 bean 执行 AbstractAutowireCapableBeanFactory#doCreateBean 流程时，会回调 InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation，触发代理对象的创建。
2. 优先级最小。@Order 注解或 Ordered 接口的约定，无优先级最低，有优先级的越小越高。

```java
public class ProxyProcessorSupport extends ProxyConfig implements Ordered, BeanClassLoaderAware, AopInfrastructureBean {
    int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
    
	private int order = Ordered.LOWEST_PRECEDENCE;
    
	@Override
	public int getOrder() {
		return this.order;
	}
}
```



![image-20240130014314907](https://image-hosting.jellyfishmix.com/20240130014320.png)



## 收集切面方法 advisor

收集切面实际收集的是 Advisor

一个 Advisor 可以视为一个切入点(join point) + 一个 advice 的结合体。具体到 aspect 切面类中的方法，方法体 + 方法上的 advice 注解可以看做一个 advisor。

### 收集 advisor

1. 找到所有 spring 原生的 advisor，不是核心逻辑，无需关注。SpringFramework 原生的 advisor 写起来太复杂了，主流的编写还是以 AspectJ 形式为主，所以无需关注，通常也没有 spring 原生的 advisor 被注册。
2. 解析 BeanFactory 中所有的 aspectj 切面，并构造 advisor。核心逻辑。

```java
    // org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator

	protected List<Advisor> findCandidateAdvisors() {
        // Add all the Spring advisors found according to superclass rules.
        // 找到所有 spring 原生的 advisor，不是核心逻辑
        List<Advisor> advisors = super.findCandidateAdvisors();
        // Build Advisors for all AspectJ aspects in the bean factory.
        // 解析 BeanFactory 中所有的 aspectj 切面，并构造 advisor。核心逻辑。
        if (this.aspectJAdvisorsBuilder != null) {
            advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
        }
        return advisors;
    }

	// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator

	protected List<Advisor> findCandidateAdvisors() {
		Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}
```

### 解析切面类构造 advisor -- buildAspectJAdvisors

这个收集 advisor 的方法，调用处位于 AnnotationAwareAspectJAutoProxyCreator 中(一个 BeanPostProcessor)，而 BeanPostProcessor 的回调处理每个 bean 时都会调用。因此使用了双检锁保证 aspectNames 切面信息和 advisor 仅收集一遍，避免重复收集带来的开销。

也就是说，收集 advisor 以懒加载的方式，当某个 bean 触发 doCreateBean，走到 BeanPostProcessor 的流程时，AnnotationAwareAspectJAutoProxyCreator 这个 BeanPostProcessor 会进行 advisor 的收集，仅收集一遍后面不会再次收集。

主要逻辑:

1. 获取 BeanFactory 中所有的 beanName
2. 借助 beanFactory 获取 bean 的 clazz, 有效避免提前创建 bean
3. 判断当前 bean 是否是一个切面类，是则获取 advisor。

```java
// org.springframework.aop.aspectj.annotation.BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new ArrayList<>();
					aspectNames = new ArrayList<>();
					// 获取 BeanFactory 中所有的 beanName
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
					for (String beanName : beanNames) {
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
						// 借助 beanFactory 获取 bean 的 clazz, 有效避免提前创建 bean
						Class<?> beanType = this.beanFactory.getType(beanName, false);
						if (beanType == null) {
							continue;
						}
						// 判断当前 bean 是否是一个切面类
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							// 下面是单实例切面类会走的流程，多实例切面类暂不关注
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
								// 获取 advisor
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new ArrayList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
```

#### 判断当前 bean 是否是一个切面类

1. 判断是否被 @Aspect 注解标注，并且不是被 aspectj 静态代理编译出的类。aspectj 静态代理编译出的类可能也有 @Aspect 注解，但 spring-aop 无法支持。

```java
	// org.springframework.aop.aspectj.annotation.AbstractAspectJAdvisorFactory#isAspect

	@Override
	public boolean isAspect(Class<?> clazz) {
		// 判断是否被 @Aspect 注解标注，并且不是被 aspectj 编译的类
		return (hasAspectAnnotation(clazz) && !compiledByAjc(clazz));
	}

	// org.springframework.aop.aspectj.annotation.AbstractAspectJAdvisorFactory#hasAspectAnnotation

	private boolean hasAspectAnnotation(Class<?> clazz) {
		return (AnnotationUtils.findAnnotation(clazz, Aspect.class) != null);
	}
```

#### 获取 advisors

1. 获取切面类的 clazz
2. 校验一下切面类上是不是标注了 @Aspect 注解
3. 逐个解析 advice 方法，并封装为 advisor
4. 通过对切面类内部加入 SyntheticInstantiationAdvisor，达到延迟初始化切面 bean 的目的
5. 对 @DeclareParent 注解功能的支持(AspectJ 的 introduction)

```java
	// org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory#getAdvisors

	@Override
	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
		// 切面类的 clazz
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
		// 校验一下切面类上是不是标注了 @Aspect 注解
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		// 此处利用 decorator(装饰者)模式，目的是保证 advisor 不会被多次实例化
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new ArrayList<>();
		// 遍历切面类候选 advisor 方法
		for (Method method : getAdvisorMethods(aspectClass)) {
			// Prior to Spring Framework 5.2.7, advisors.size() was supplied as the declarationOrderInAspect
			// to getAdvisor(...) to represent the "current position" in the declared methods list.
			// However, since Java 7 the "current position" is not valid since the JDK no longer
			// returns declared methods in the order in which they are declared in the source code.
			// Thus, we now hard code the declarationOrderInAspect to 0 for all advice methods
			// discovered via reflection in order to support reliable advice ordering across JVM launches.
			// Specifically, a value of 0 aligns with the default value used in
			// AspectJPrecedenceComparator.getAspectDeclarationOrder(Advisor).
            // 收集被特定 advice 注解标注的 advisor
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 0, aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		// 通过在装饰者内部的开始加入 SyntheticInstantiationAdvisor 增强器，达到延迟初始化切面 bean 的目的
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		// 对 @DeclareParent 注解功能的支持(AspectJ 的 introduction)
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}
```

#### 根据候选方法获取 advisor

1. 尝试获取候选方法对应的 pointcut 注解，如果没有说明此方法非 pointcut
2. 构造 pointcut 表达式模型

```java
	// org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory#getAdvisor

	@Override
	@Nullable
	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());

		// 尝试获取候选方法对应的 pointcut 表达式，如果没有说明此方法非 pointcut
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}

		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}

	// org.springframework.aop.aspectj.annotation.ReflectiveAspectJAdvisorFactory#getPointcut

	@Nullable
	private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
		// 尝试获取候选方法对应的 pointcut 注解，如果没有说明此方法非 pointcut
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// 构造 pointcut 表达式模型
		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		if (this.beanFactory != null) {
			ajexp.setBeanFactory(this.beanFactory);
		}
		return ajexp;
	}

	private static final Class<?>[] ASPECTJ_ANNOTATION_CLASSES = new Class<?>[] {
			Pointcut.class, Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class};

	@Nullable
	protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
		for (Class<?> clazz : ASPECTJ_ANNOTATION_CLASSES) {
			AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) clazz);
			if (foundAnnotation != null) {
				return foundAnnotation;
			}
		}
		return null;
	}
```



## TargetSource -- target 的封装

spring-aop 中，代理对象 proxy 并没有直接代理 target ，而是对 target 封装了一层 TargetSource:

![img](https://image-hosting.jellyfishmix.com/20240201000239.png)

```java
public interface TargetSource extends TargetClassAware {
	Class<?> getTargetClass();
	boolean isStatic();
	Object getTarget() throws Exception;
	void releaseTarget(Object target) throws Exception;
}
```

### 封装一层 TargetSource 的原因

1. SingletonTargetSource: 每次调用 getTarget 都返回同一个 target bean 实例。与直接代理 target 无任何区别。

2. PrototypeTargetSource: 每次调用 getTarget 都会从 BeanFactory 中创建一个全新的 target bean (被它封装的 target bean 必须为原型 bean)

3. CommonsPool2TargetSource: 内部维护了一个对象池，每次 getTarget 时从对象池中取(底层使用 apache 的 ObjectPool)

4. ThreadLocalTargetSource: 每次调用 getTarget 都会从 ThreadLocal 中获取 target bean，线程唯一(被它封装的 target bean 必须为原型 bean))

5. HotSwappableTargetSource: 内部维护了一个可以热替换的目标对象引用，每次调用 getTarget 时都返回它。它提供了一个线程安全的 swap 方法，以热替换 TargetSource 中被代理的目标对象。



## 创建代理

### spring 内部代理创建的位置 -- 无需关注

1. 注意 InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation 的回调不是真正创建代理的位置，仅用于 spring 内部一些代理的创建。为了便于理解可以暂不关注。
2. 业务创建代理对象 proxy 的位置在 BeanPostProcessor#postProcessAfterInitialization

```java
	// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessBeforeInstantiation

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

        // 决定是否要提前增强当前 bean
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
            // 被增强过的 bean 不会再次被增强
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
            // 基础类型的 bean 不会被提前增强，被跳过的 bean 不会被提前增强
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
        // 获取代理对象的装饰者 TargetSource
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}
```

### 触发为目标 bean 创建代理 -- postProcessAfterInitialization 回调

1. 这是 BeanPostProcessor 提供的回调方法，是创建代理对象 proxy 的位置。调用 wrapIfNecessary，创建代理如果有必要。

```java
	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                // 核心，创建代理如果有必要
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

### 核心，创建代理如果有必要 -- wrapIfNecessary

1. 根据 bean 的属性，判断目标 bean 是否需要创建代理。
2. 获取目标 bean 可用的 advisor，如果存在可用 advisor，则创建代理，固定用 SingletonTargetSource 封装 target bean。

```java
	// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#wrapIfNecessary

	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        // 根据 bean 的属性，判断目标 bean 是否需要创建代理
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		// 如果上面的判断都没有成立，则尝试进行代理对象的创建
		// 获取目标 bean 可用的 advisor
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			// 创建代理，固定用 SingletonTargetSource 封装 target bean
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

### 获取可用的 advisor -- getAdvicesAndAdvisorsForBean

```java
	@Override
	@Nullable
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

		// 核心方法，获取可用的 advisor
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
```

### 获取可用的 advisor -- findEligibleAdvisors

1. 获取已收集的所有 advisor。
2. 筛选出可以织入当前 bean 的 advisor，并添加额外的 advisor。
3. 如果有可用的 advisor 则排序。

```java
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		// 获取已收集的所有 advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		// 筛选出可以织入当前 bean 的 advisor
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		// 添加额外的 advisor
		extendAdvisors(eligibleAdvisors);
		// 如果有可用的 advisor
		if (!eligibleAdvisors.isEmpty()) {
			// advisor 排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}

	// org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#findAdvisorsThatCanApply

	protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
			// 核心方法
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}
```

### 筛选出可以织入目标 bean 的 advisor

1. 遍历正常的的 advisor，判断当前候选 advisor 是否适用目标 bean

```java
	// org.springframework.aop.support.AopUtils#findAdvisorsThatCanApply

	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new ArrayList<>();
		// introduction 引介 advisor, 很少用到引介，暂不关注
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		// 遍历正常的的 advisor
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			// 判断当前候选 advisor 是否适用指定的 bean
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}
```

### 判断候选 advisor 是否适用目标 bean -- canApply

1. 根据 pointcut 判断 advisor 是否适用
   1. 判断 target 类是否与 advisor 织入目标类符合
   2. 逐个判断 target 类声明的每个方法，是否能被当前 pointcut 织入，存在可用织入则立即返回 true

```java
	// org.springframework.aop.support.AopUtils#canApply(org.springframework.aop.Pointcut, java.lang.Class<?>, boolean)

	public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		// introduction 引介 advisor 相关，暂不关注
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		// 方法切入点的增强器匹配逻辑
		else if (advisor instanceof PointcutAdvisor) {
			// 根据 pointcut 判断 advisor 是否适用
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			// 收集到的 advisor 没有 pointcut，默认适用(这里是兜底逻辑，理论上之前收集的 advisor 都有 pointcut)
			return true;
		}
	}
	
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		// 判断 target 类是否与 advisor 织入目标类符合
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		// introduction 引介相关，暂不关注
		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			// 逐个判断 target 类声明的每个方法，是否能被当前 pointcut 织入，存在可用织入则立即返回 true
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}
```

### 添加额外的 advisor extendAdvisors -- extendAdvisors

不重要，暂不关注

### 创建代理对象 proxy -- createProxy

1. 记录被代理 bean 的原始类型
2. ProxyFactory 的初始化。
3. 为使用 cglib 或 jdk 动态代理进行增强预先作准备。
4. 构建所有的 advisor(大部分 advisor 是创建代理 proxy 前获取可用 advisor 时直接拿到的)。
5. 通过 proxyFactory 创建 proxy 对象。

```java
	// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#createProxy

	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		// 记录被代理 bean 的原始类型
		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		// 代理工厂的初始化
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		// 为使用 cglib 或 jdk 动态代理进行增强预先作准备
		if (proxyFactory.isProxyTargetClass()) {
			// Explicit handling of JDK proxy targets (for introduction advice scenarios)
			// 使用 jdk 动态代理进行增强
			if (Proxy.isProxyClass(beanClass)) {
				// Must allow for introductions; can't just set interfaces to the proxy's interfaces only.
				for (Class<?> ifc : beanClass.getInterfaces()) {
					proxyFactory.addInterface(ifc);
				}
			}
		}
		else {
			// No proxyTargetClass flag enforced, let's apply our default checks...
			// 使用 cglib 进行增强
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		// 构建所有的 advisor(大部分 advisor 是创建代理 proxy 前获取可用 advisor 时直接拿到的)
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		// 通过 proxyFactory 创建 proxy 对象
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```

### 根据入参构建 advisors -- buildAdvisors

大部分 advisor 是创建代理 proxy 前获取可用 advisor 时直接拿到的。暂不关注此方法 buildAdvisors

1. 整合早期 spring 原生 aop 的逻辑，无需关注。
2. 包装出 advisor。

```java
	// org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#buildAdvisors

	protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
		// Handle prototypes correctly...
		Advisor[] commonInterceptors = resolveInterceptorNames();

		List<Object> allInterceptors = new ArrayList<>();
		// 整合早期 spring 原生 aop 的逻辑，无需关注
		if (specificInterceptors != null) {
			if (specificInterceptors.length > 0) {
				// specificInterceptors may equals PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS
				allInterceptors.addAll(Arrays.asList(specificInterceptors));
			}
			if (commonInterceptors.length > 0) {
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isTraceEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.trace("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}

		Advisor[] advisors = new Advisor[allInterceptors.size()];
		// 包装出 advisor，无需关注
		for (int i = 0; i < allInterceptors.size(); i++) {
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}
```

### 通过 proxyFactory 创建 proxy 对象 -- ProxyFactory#getProxy

```java
	// org.springframework.aop.framework.ProxyFactory#getProxy(java.lang.ClassLoader)
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
```

### 判断使用 jdk 动态代理还是 cglib 动态代理

```java
	// org.springframework.aop.framework.ProxyCreatorSupport#createAopProxy

	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}

	// org.springframework.aop.framework.DefaultAopProxyFactory#createAopProxy

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			// 如果要代理的 target 本身就是接口，或者已经是被 jdk 动态代理了的代理对象，则使用 jdk 动态代理
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			// 否则使用 cglib 动态代理
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

### jdk 动态代理和 cglib 动态代理各自创建代理的实现

org.springframework.aop.framework.AopProxy#getProxy(java.lang.ClassLoader)

决定好代理方式后，jdk 动态代理和 cglib 动态代理各有自己创建代理的实现。

org.springframework.aop.framework.JdkDynamicAopProxy#getProxy(java.lang.ClassLoader)

org.springframework.aop.framework.CglibAopProxy#getProxy(java.lang.ClassLoader)

![image-20240201010010382](https://image-hosting.jellyfishmix.com/20240201010010.png)

### jdk 动态代理创建 proxy 的实现

1. 直接调用了 jdk 动态代理的 api  java.lang.reflect.Proxy#newProxyInstance

```java
	// org.springframework.aop.framework.JdkDynamicAopProxy#getProxy(java.lang.ClassLoader)

	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isTraceEnabled()) {
			logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		// jdk 动态代理的 api
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```

### cglib 动态代理创建 proxy 的实现

cglib 动态代理创建 proxy 的实现逻辑较复杂暂不深入了。



## 无法被 spring-aop 机制代理的 bean

spring-aop 机制需要 AbstractAutoProxyCreator(一个 BeanPostProcessor)。收集切面类、创建代理对象的时机也是 BeanPostProcessor 的回调方法。

ioc 容器 ApplicationContext 刷新(初始化)的位置是 AbstractApplicationContext#refresh。此方法中创建 BeanPostProcessor 并添加进 beanFactory 的位置是 AbstractApplicationContext#registerBeanPostProcessors。

早于 BeanPostProcessor 创建时机前已经创建的 bean，则无法被 spring-aop 机制代理。例如 AbstractApplicationContext#refresh 方法中比 registerBeanPostProcessors 方法调用时机更早的 invokeBeanFactoryPostProcessors 方法中，用到的 BeanDefinitionRegistryPostProcessor, BeanFactoryPostProcessor 这样的 bean，无法被 spring-aop 机制代理。

![Screen Shot 2024-02-01 at 01.40.40](https://image-hosting.jellyfishmix.com/20240201014256.png)
