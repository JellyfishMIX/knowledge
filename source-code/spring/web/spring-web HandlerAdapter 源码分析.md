# spring-web HandlerAdapter 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## HandlerAdapter 接口

提供作为处理器适配器的能力。

1. supports 方法判断是否支持该 handler。

```java
public interface HandlerAdapter {
	/**
	 * 判断是否支持该 handler
	 */
	boolean supports(Object handler);

	/**
	 * handler 执行，返回 ModelAndView 结果
	 */
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;

	/**
	 * 返回请求的最新更新时间，如果不支持该操作，则返回 -1 即可
	 */
	long getLastModified(HttpServletRequest request, Object handler);
}
```

### 类层次

![img](https://image-hosting.jellyfishmix.com/20230126015751.png)

### HandlerAdapter 的定位/职责

HandlerAdapter 存在的原因是，springMVC 通过 HandlerMapping 为请求找到合适的 HandlerExecutionChain 处理器执行链，包含处理器(handler)和拦截器(interceptors)。其中处理器(handler)的实现有多种，例如实现 Controller 接口，实现 HttpRequestHandler 接口，或者使用 @RequestMapping 注解将方法作为一个处理器等。导致 springMVC 无法直接执行这个处理器，所以这里需要一个适配器(adapter)，由它去执行处理器(handler)。

HandlerAdapter 也有多种，哪种 adapter 支持处理这种类型的 handler，则由该 adapter 去执行，如下:

- HttpRequestHandlerAdapter: 适配实现了 HttpRequestHandler 接口的处理器。
- SimpleControllerHandlerAdapter: 适配实现了 Controller 接口的处理器。
- SimpleServletHandlerAdapter: 适配实现了 Servlet 接口的处理器。
- RequestMappingHandlerAdapter: 适配 HandlerMethod 类型的处理器，也就是通过 @RequestMapping 注解标注的方法。

重点关注 RequestMappingHandlerAdapter，这种方式是目前使用最普遍的。



## HandlerAdapter 初始化过程

DispatcherServlet 重写 FrameworkServlet#onRefresh 方法，调用 DispatcherServlet#initHandlerAdapters 方法，初始化 HandlerAdapter。

org.springframework.web.servlet.DispatcherServlet#onRefresh

```java
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```

### DispatcherServlet#initHandlerAdapters 方法

org.springframework.web.servlet.DispatcherServlet#initHandlerAdapters

1. 如果探测功能开启，则扫描已注册的 HandlerAdapter 的 beans，添加到 handlerAdapters 集合中，默认开启。这里会进行排序，可以通过实现 Order 接口设置排序值。

2. 如果探测功能关闭，则获得 beanName 为 "handlerAdapter" 对应的 bean，添加至 handlerAdapters。

3. 未获取到 handlerAdapter 则获取默认的，如下: HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter, RequestMappingHandlerAdapter

   ```properties
   org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
   ```



```java
	private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;

		// 如果探测功能开启，则扫描已注册的 HandlerAdapter 的 beans，添加到 handlerAdapters 集合中，默认开启。这里会进行排序，可以通过实现 Order 接口设置排序值。
		if (this.detectAllHandlerAdapters) {
			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<>(matchingBeans.values());
				// We keep HandlerAdapters in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
		}
		// 如果探测功能关闭，则获得 beanName 为 "handlerAdapter" 对应的 bean，添加至 handlerAdapters
		else {
			try {
				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
				this.handlerAdapters = Collections.singletonList(ha);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerAdapter later.
			}
		}

		// Ensure we have at least some HandlerAdapters, by registering
		// default HandlerAdapters if no other adapters are found.
		// 未获取到 handlerAdapter 则获取默认的
		if (this.handlerAdapters == null) {
			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
			if (logger.isTraceEnabled()) {
				logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
						"': using default strategies from DispatcherServlet.properties");
			}
		}
	}
```



## AbstractHandlerMethodAdapter#handle 方法 -- 处理请求

org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter#handle

处理请求，封装了一层内部调用，具体逻辑 handleInternal 交由子类实现。由 DispatcherServlet#doDispatch 调用。

```java
	@Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		return handleInternal(request, response, (HandlerMethod) handler);
	}
```



## 三个默认的 HandlerAdapter

org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter

对于 HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter，阅读 HandlerAdapter 接口的定义后，没有太多需要说明的。我们重点关注 RequestMappingHandlerAdapter。

```java
public class HttpRequestHandlerAdapter implements HandlerAdapter {
	@Override
	public boolean supports(Object handler) {
		return (handler instanceof HttpRequestHandler);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}
}

public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```



## RequestMappingHandlerAdapter

### RequestMappingHandlerAdapter#afterPropertiesSet 方法 -- 初始化 adapter

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#afterPropertiesSet

RequestMappingHandlerAdapter 实现了 InitializingBean 接口，在 spring-ioc 初始化 RequestMappingHandlerAdapter 的 bean 实例时候，会调用此方法，完成一些初始化工作。

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware, InitializingBean {
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		// 初始化 ControllerAdvice 相关
		initControllerAdviceCache();

		// 初始化 argumentResolvers 属性
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		// 初始化 initBinderArgumentResolvers 属性
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		// 初始化 returnValueHandlers 属性
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
}
```

#### @ControllerAdvice 注解

用于 Controller 类的增强，@ControllerAdvice 加载增强类上，此类中可使用方法级别的注解增强方法，例如 @ExceptionHandler 注解加载方法上，用于处理 Controller 抛出的异常。demo:

```java
@Order(Integer.MAX_VALUE)
@ControllerAdvice
public class DefaultExceptionHandler {
    protected Logger log = LoggerFactory.getLogger(DefaultExceptionHandler.class);

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public ResponseEntity defaultErrorHandler(Exception e) {
        // ...
        return xxx;
    }
}
```

### RequestMappingHandlerAdapter#getDefaultArgumentResolvers 方法 -- 获取默认的参数解析器

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getDefaultArgumentResolvers

获取默认的参数解析器，按顺序添加了多个参数解析器对象。

```java
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```

### RequestMappingHandlerAdapter#getDefaultInitBinderArgumentResolvers 方法 -- 获取默认的参数绑定器

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getDefaultInitBinderArgumentResolvers

获取默认的参数绑定器，按顺序添加了多个参数解析器对象。

```java
	private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(20);

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));

		return resolvers;
	}
```

### RequestMappingHandlerAdapter#getDefaultReturnValueHandlers 方法 -- 获取默认的返回值处理器

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getDefaultReturnValueHandlers

获取默认的返回值处理器，按顺序添加了多个返回值处理器对象。

```java
	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>(20);

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(),
				this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
				this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		}
		else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```

### RequestMappingHandlerAdapter#handleInternal 方法 -- 处理请求

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#handleInternal

处理请求，由 AbstractHandlerMethodAdapter#handle 方法触发。

```java
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		ModelAndView mav;
		// 校验请求(requestMethod 和 pre-session 的校验)
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		// 调用 HandlerMethod 方法，同步相同 Session 的逻辑，默认情况 false
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				// 获取 session 的锁对象
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```

### RequestMappingHandlerAdapter#invokeHandlerMethod

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#invokeHandlerMethod

处理请求，由 RequestMappingHandlerAdapter#handleInternal 方法触发。

1. 创建 ServletWebRequest 对象，包含了 request 和 response。创建 WebDataBinderFactory 对象，和数据绑定相关。创建 ModelFactory 对象，和往 Model 对象设置数据相关。创建 ServletInvocableHandlerMethod 对象，ModelAndViewContainer 对象，AsyncWebRequest 对象，WebAsyncManager 对象，并初始其相关属性。
2. 核心，调用 ServletInvocableHandlerMethod#invokeAndHandle 方法处理请求。
3. 获得 ModelAndView 对象，标记请求完成。

```java
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
		// 创建 ServletWebRequest 对象，包含了 request 和 response
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			// 创建 WebDataBinderFactory 对象，和数据绑定相关
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			// 创建 ModelFactory 对象，和往 Model 对象设置数据相关
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			// 创建 ServletInvocableHandlerMethod 对象，并设置其相关属性
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			// 创建 ModelAndViewContainer 对象，并初始其相关属性
			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			// 创建 AsyncWebRequest 异步请求对象
			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			// 创建 WebAsyncManager 异步请求管理器对象
			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			// 异步处理，并发结果相关
			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			// 核心，由 ServletInvocableHandlerMethod 处理请求
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			// 获得 ModelAndView 对象
			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			// 标记请求完成
			webRequest.requestCompleted();
		}
	}
```

