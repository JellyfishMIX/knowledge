# spring-aop 代理对象 proxy 的执行



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## Cglib 动态代理 proxy 对象方法执行的入口

我们以 CglibAopProxy 动态代理举例。proxy 被调用时，会触发 DynamicAdvisedInterceptor#intercept。

1. DynamicAdvisedInterceptor 实现的接口是 org.aopalliance.intercept.MethodInterceptor。本质也是一个 org.aopalliance.aop.Advice

![image-20240201011834375](https://image-hosting.jellyfishmix.com/20240201011834.png)



## DynamicAdvisedInterceptor 增强逻辑

动态代理对象的 advice 增强核心步骤分为三步骤:

1. 获取 MethodInterceptor 责任链
2. 构建方法执行器 CglibMethodInvocation
3. 执行 MethodInterceptor 责任链和 targetMethod

```java
// org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept

public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();
    try {
		// 如果有在 @EnableAspectJAutoProxy 注解上配置 exposeProxy 属性为 true，
        // 则会把当前代理对象放入 aop 上下文中
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // 从 TargetSource 中取出目标对象
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        // 根据当前 targetMethod，获取要执行的增强器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        
        // 如果没有要执行的增强器，则直接执行目标方法
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // 否则，构造增强器链，执行增强器的逻辑
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    } // finally ......
}
```



## 获取 MethodInterceptor 责任链

### 带缓存地获取 MethodInterceptor 责任链 AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice

1. 使用了缓存的设计，target method 的 advicor 第一次创建后，以后就直接从缓存中读了，不需要多次获取和解析。

```java
// org.springframework.aop.framework.AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice

public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        // 核心逻辑
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```

在循环处理增强器之前，这里先进行了一些基本的前置处理，这里它初始化了一个 AdvisorAdapterRegistry，意为增强器适配器的注册器，它的主要作用是将 AspectJ 类型的增强器，转换为 MethodInterceptor(AOP 联盟的那个 MethodInterceptor)并返回。此举的目的，会在接下来的 CglibMethodInvocation 中得以体现，咱这里先有个印象即可。

### 获取 MethodInterceptor 责任链 -- DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice

AdvisorAdapterRegistry 用于将 Advisor 转换为 MethodInterceptor(AOP 联盟约定的 MethodInterceptor)。MethodInterceptor 在 CglibMethodInvocation 中会用到。

1. 获取 AdvisorAdapterRegistry 实例, AdvisorAdapterRegistry 用于将 Advisor 转换为 MethodInterceptor(AOP 联盟规范约定的 MethodInterceptor)。
2. 遍历 advisor
   1. 获取 MethodMatcher, 用于根据当前 advisor 的 pointCut 表达式, 判断当前 advisor 是否和 targetMethod 匹配
   2. 当前接入点(targetMethod)是否匹配当前 advisor
   3. 匹配后，使用 AdvisorAdapterRegistry 将 Advisor 转换为 MethodInterceptor。向结果集中添加 MethodInterceptor, 按照 MethodMatcher 动态和静态两种处理:
      1. 动态 MethodMatcher 不会直接添加进结果集，而是封一层运行时匹配入参的 InterceptorAndDynamicMethodMatcher，再添加进结果集。
      2. 静态 MethodMatcher 直接添加进结果集。
3. 返回和 targetMethod 匹配的 MethodInterceptor 责任链。

```java
// org.springframework.aop.framework.DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice

	@Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {
		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		// // 获取 AdvisorAdapterRegistry 实例(AdvisorAdapterRegistry 用于将 Advisor 转换为 MethodInterceptor, MethodInterceptor 在 CglibMethodInvocation 中会用到)
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		Advisor[] advisors = config.getAdvisors();
		List<Object> interceptorList = new ArrayList<>(advisors.length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		Boolean hasIntroductions = null;

		// 遍历 advisor
		for (Advisor advisor : advisors) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					// 获取 MethodMatcher, 用于根据当前 advisor 的 pointCut 表达式, 判断当前 advisor 是否和 targetMethod 匹配
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					boolean match;
					// introduction 引介, 不关注
                    // ...
					// 当前接入点(targetMethod)是否匹配当前 advisor
					else {
						match = mm.matches(method, actualClass);
					}
					// 如果 targetMethod 匹配当前 advisor
					if (match) {
						// 使用 AdvisorAdapterRegistry 将 Advisor 转换为 MethodInterceptor
						MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
						// 动态匹配器不会直接添加 MethodInterceptor，而是封一层运行时匹配入参的 InterceptorAndDynamicMethodMatcher
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						// 静态匹配器直接添加 MethodInterceptor
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			// introduction 引介，不关注
            // ...
		}

		// 返回和 targetMethod 匹配的 MethodInterceptor 责任链
		return interceptorList;
	}
```



## 构建方法执行器 CglibMethodInvocation

CglibMethodInvocation 的声明，可以看到本质是一个 MethodInvocation, proceed 方法用于执行 MethodInterceptor 责任链和 targetMethod

```java
private static class CglibMethodInvocation extends ReflectiveMethodInvocation
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable
public interface ProxyMethodInvocation extends MethodInvocation
```



## 执行 MethodInterceptor 责任链

```java
		// org.springframework.aop.framework.CglibAopProxy.CglibMethodInvocation#proceed

		@Override
		@Nullable
		public Object proceed() throws Throwable {
			return super.proceed();
		}
```

### proceed 方法 -- 执行 MethodInterceptor 责任链并调用 targetMethod

1. proceed 方法是递归，没有 MethodInterceptor 或 MethodInterceptor 全部执行完毕后，会执行 targetMethod
2. 取出本次递归迭代的 MethodInterceptor
   1. 动态 MethodInterceptor 与当前入参匹配则执行 MethodInterceptor, 与当前入参不匹配, 递归迭代下一个 MethodInterceptor。
   2. 静态 MethodInterceptor 调用 invoke 方法执行 advice 逻辑，注意需要手动在 MethodInterceptor 实现中调用 proceed 方法进行下一次递归迭代。

```java
	// org.springframework.aop.framework.ReflectiveMethodInvocation#proceed

	@Override
	@Nullable
	public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			// 没有 MethodInterceptor 或 MethodInterceptor 全部执行完毕后，会执行 targetMethod
			return invokeJoinpoint();
		}

		// 取出本次递归的 MethodInterceptor
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		// 动态 MethodInterceptor
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			// 与当前入参匹配则执行 MethodInterceptor
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				// 与当前入参不匹配, 递归迭代下一个 MethodInterceptor
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			// 静态 MethodInterceptor 执行 advice
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```



### 执行 advice 增强逻辑 -- MethodInterceptor#invoke

1. 不同的 Advice 类型的 MethodInterceptor 会调用不同的 Advice 声明方法，然后调用 MethodInterceptor 进行下次递归迭代。

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {
	private final MethodBeforeAdvice advice;

	/**
	 * Create a new MethodBeforeAdviceInterceptor for the given advice.
	 * @param advice the MethodBeforeAdvice to wrap
	 */
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		return mi.proceed();
	}
}
```



## JdkDynamicAopProxy

1. 根据当前 targetMethod，获取要执行的 MethodInterceptor 责任链。
2. 执行 ReflectiveMethodInvocation#proceed 方法，上文已经分析过。因此可以观察到 jdk 动态代理和 cglib 动态代理，执行 MethodInterceptor 责任链和调用 targetMethod 的逻辑是统一的。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        // equals 方法不代理
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            // The target does not implement the equals(Object) method itself.
            return equals(args[0]);
        }
        // hashCode 方法不代理
        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            // The target does not implement the hashCode() method itself.
            return hashCode();
        }
        // 方法来自于 DecoratingProxy 接口的，也不代理
        else if (method.getDeclaringClass() == DecoratingProxy.class) {
            // There is only getDecoratedClass() declared -> dispatch to proxy config.
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }
        // 目标对象本身就是实现了Advised接口，也不代理
        else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            // Service invocations on ProxyConfig with the proxy config...
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal;

        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // 根据当前 targetMethod，获取要执行的 MethodInterceptor 责任链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        if (chain.isEmpty()) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            MethodInvocation invocation =
                    new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // 构造增，执行增强器的逻辑
            retVal = invocation.proceed();
        }
        // 返回值的处理 ......
        return retVal;
    } // finally ......
}
```



## Aspect 中的四种通知在源码中的实现

### @Before

1. 先执行 Advice，再执行 targetMethod。

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
	private MethodBeforeAdvice advice;

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
        // 先执行前置通知，再执行目标方法
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis() );
		return mi.proceed();
	}
}
```

### @After

1. 执行 targetMethod 后，在 finally 中执行 Advice 方法

```java
public class AspectJAfterAdvice extends AbstractAspectJAdvice
		implements MethodInterceptor, AfterAdvice, Serializable {

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		finally {
            // 执行目标方法后，在 finally 中执行后置方法
			invokeAdviceMethod(getJoinPointMatch(), null, null);
		}
	}
}
```

### @AfterReturning

1. 不写 try-catch，说明不出现任何异常时才会触发 Advice 方法。

```java
public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {
	private final AfterReturningAdvice advice;

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
        // 不写 try-catch，说明不出现任何异常时才会触发 AfterReturning 通知。
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
}
```

### @AfterThrowing

1. try-catch 中写  Advice 方法，触发异常时才会执行  Advice 方法。

```java
public class AspectJAfterThrowingAdvice extends AbstractAspectJAdvice
		implements MethodInterceptor, AfterAdvice, Serializable {

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
            // try-catch 中写 AfterThrowingAdvice，触发异常时才会执行 AfterThrowingAdvice。
			if (shouldInvokeOnThrowing(ex)) {
				invokeAdviceMethod(getJoinPointMatch(), null, ex);
			}
			throw ex;
		}
	}
}
```

### @Around

1. 获取调用 advice 方法需要的参数。
2. 使用反射机制调用 advice 方法和 targetMethod，通过编排 advice 方法和 targetMethod 的顺序控制 around。

```java
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		if (!(mi instanceof ProxyMethodInvocation)) {
			throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
		}
		ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
		ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
		JoinPointMatch jpm = getJoinPointMatch(pmi);
		return invokeAdviceMethod(pjp, jpm, null, null);
	}

	protected Object invokeAdviceMethod(JoinPoint jp, @Nullable JoinPointMatch jpMatch,
			@Nullable Object returnValue, @Nullable Throwable t) throws Throwable {
        // 1. 获取调用 advice 方法需要的参数
        // 2. 使用反射机制调用 advice 方法
		return invokeAdviceMethodWithGivenArgs(argBinding(jp, jpMatch, returnValue, t));
	}

	protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
		Object[] actualArgs = args;
		if (this.aspectJAdviceMethod.getParameterCount() == 0) {
			actualArgs = null;
		}
		try {
			ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
			// TODO AopUtils.invokeJoinpointUsingReflection
			return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
		}
		catch (IllegalArgumentException ex) {
			throw new AopInvocationException("Mismatch on arguments to advice method [" +
					this.aspectJAdviceMethod + "]; pointcut expression [" +
					this.pointcut.getPointcutExpression() + "]", ex);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
```



## tips

### 责任链节点递归 vs 责任链节点遍历-- 个人观点

#### 两种写法案例

1. spring-aop ReflectiveMethodInvocation 执行 proceed 方法来执行 MethodInterceptor 责任链，用的是递归的写法。由每个 MethodInterceptor 自行决定是否要执行 proceed 方法来迭代下次递归。
2. 和 tomcat 的 fitler 递归写法一样，由 filter 自行决定是否递归迭代下个 filter, 只有责任链上所有候选节点都执行完，才能调用 targetMethod
3. 个人认为递归写法心智负担有点高，不过也没办法，递归写法能省返回值坑位。如果是遍历责任链节点的写法，需要责任链上每个节点返回值 boolean 表示是否进行下次遍历迭代，占用了返回值的位置。
4. 责任链遍历的写法典型代表 BeanDefinitionRegistryPostProcessor, BeanFactoryPostProcessor BeanPostProcessor。

#### 两种写法区别

1. 没有大区别，只是写法不同。主要目的拦截干预，做成递归好一些。让每个节点都获取到 target 做处理，用遍历写法好一些。
2. 递归的好处: 调用栈明确，在某个责任链节点调用时，能看到之前节点递归的调用栈。递归的弊端: 心智负担高一些，责任链节点的实现需要注意显式调用下次递归迭代，否则责任链无法继续迭代。
3. 遍历写法的好处: 遍历过程中打断点找触发拦截的节点很方便，便于排查问题。

### MethodInterceptor 名称的含义

1. 在调用 targetMethod 前，遍历责任链，这样的行为的确是拦截了方法，叫 MethodInterceptor 挺合理。
2. jdk 通常对于这样的行为，都叫 intercept
3. 其实叫 filter 叫 intercept 都行，tomcat 就叫 filter，看习惯
