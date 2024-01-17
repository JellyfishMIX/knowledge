# spring-web HandlerMethodReturnValueHandler 源码分析



## HandlerMethodReturnValueHandler 的使用处，处理请求

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle

HandlerMethodReturnValueHandler 在处理请求时使用，以下是使用处 ServletInvocableHandlerMethod#invokeAndHandle 方法。

1. 执行调用处理请求，设置 responseStatus。
2. 如果 returnValue 为空，设置 ModelAndViewContainer 为请求已处理然后 return，和 @ResponseStatus 注解相关。
3. 设置 ModelAndViewContainer 为请求未处理，通过 HandlerMethodReturnValueHandlerComposite(HandlerMethodReturnValueHandler) 处理返回值。

```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
    @Nullable
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
    
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		// 执行调用处理请求
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		// 设置 responseStatus
		setResponseStatus(webRequest);

		// 如果 returnValue 为空，设置 ModelAndViewContainer 为请求已处理然后 return，和 @ResponseStatus 注解相关
		if (returnValue == null) {
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				disableContentCachingIfNecessary(webRequest);
				mavContainer.setRequestHandled(true);
				return;
			}
		}
		else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		// 设置 ModelAndViewContainer 为请求未处理
		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
			// 通过 HandlerMethodReturnValueHandlerComposite(HandlerMethodReturnValueHandler) 处理返回值
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}
}
```



## 类层次

![img](https://image-hosting.jellyfishmix.com/20230217203345.png)

返回类型有很多，因此有很多 HandlerMethodReturnValueHandler 的实现类，上图仅列出了本文会分析的两个实现类。



## HandlerMethodReturnValueHandler 接口

```java
public interface HandlerMethodReturnValueHandler {
	/**
	 * 是否支持处理当前返回类型
	 */
	boolean supportsReturnType(MethodParameter returnType);

	/**
	 * 处理当前返回值
	 */
	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;
}
```



## ModelAndViewContainer

org.springframework.web.method.support.ModelAndViewContainer

作为 Model 和 View 的容器。

```java
public class ModelAndViewContainer {
	/**
	 * 是否在 redirect 重定向时，忽略 {@link #redirectModel} 属性
	 */
	private boolean ignoreDefaultModelOnRedirect = false;

	/**
	 * 视图，Object 类型
	 */
	@Nullable
	private Object view;

	/**
	 * 默认使用的 Model，实际上是个 Map
	 */
	private final ModelMap defaultModel = new BindingAwareModelMap();

	/**
	 * redirect 重定向的 Model，在重定向时使用。
	 */
	@Nullable
	private ModelMap redirectModel;

	/**
	 * 处理器返回 redirect 视图的标识
	 */
	private boolean redirectModelScenario = false;

	/**
	 * http 响应状态
	 */
	@Nullable
	private HttpStatus status;

	private final Set<String> noBinding = new HashSet<>(4);

	private final Set<String> bindingDisabled = new HashSet<>(4);

	/**
	 * 用于设置 SessionAttribute 的标识
	 */
	private final SessionStatus sessionStatus = new SimpleSessionStatus();

	/**
	 * 请求是否处理完的标识
	 */
	private boolean requestHandled = false;
}
```

### ModelAndViewContainer#getModel 方法 -- 获取 model

org.springframework.web.method.support.ModelAndViewContainer#getModel

```java
	public ModelMap getModel() {
		// 是否使用默认 Model
		if (useDefaultModel()) {
			return this.defaultModel;
		}
		else {
			if (this.redirectModel == null) {
				this.redirectModel = new ModelMap();
			}
			return this.redirectModel;
		}
	}
```

### requestHandled 属性 -- 请求是否处理完的标识

requestHandled 主要的修改位置:

1. ServletInvocableHandlerMethod#invokeAndHandle 方法中，会先设置为 false，表示请求还未处理，再交由 HandlerMethodReturnValueHandler 结果处理器去处理。
2. 实现 @Responsebody 注解机制的 RequestResponseBodyMethodProcessor#handleReturnValue 方法中会设置为 true。

请求处理完成获得返回值 returnValue 后，接下来 RequestMappingHandlerAdapter 需要通过 ModelAndViewContainer 获取 ModelAndView 对象，会用到 requestHandled 这个属性。

### RequestMappingHandlerAdapter#getModelAndView 方法 -- 获取 model 和 view

org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#getModelAndView

1. 情况1, 如果 requestHandled 为 true，即请求已处理，则返回 null。这是例如 @Responsebody 注解不返回 ModelAndView 的原因。
2. 情况2, 如果 mavContainer 未处理，则基于 mavContainer 生成 modelAndView。

```java
	@Nullable
	private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

		modelFactory.updateModel(webRequest, mavContainer);
		// 情况1, 如果 requestHandled 为 true，即请求已处理，则返回 null
		if (mavContainer.isRequestHandled()) {
			return null;
		}
		// 情况2, 如果 mavContainer 未处理，则基于 mavContainer 生成 modelAndView
		ModelMap model = mavContainer.getModel();
		// 创建 modelAndView 对象，并设置相关属性
		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
		if (!mavContainer.isViewReference()) {
			mav.setView((View) mavContainer.getView());
		}
		if (model instanceof RedirectAttributes) {
			Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
			HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
			if (request != null) {
				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
			}
		}
		return mav;
	}
```



## HandlerMethodReturnValueHandlerComposite -- 复合体

org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite

使用了组合模式，承载着 HandlerMethodReturnValueHandler 的集合。

```java
public class HandlerMethodReturnValueHandlerComposite implements HandlerMethodReturnValueHandler {
	/**
	 * 承载着 HandlerMethodReturnValueHandler 的集合
	 */
	private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<>();
}
```

### HandlerMethodReturnValueHandlerComposite#handleReturnValue 方法 -- 处理返回值

org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite#handleReturnValue

1. 获取适合的 HandlerMethodReturnValueHandler。
2. 调用具体的 HandlerMethodReturnValueHandler 处理返回值。

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		// 获取适合的 HandlerMethodReturnValueHandler
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
        // 调用具体的 HandlerMethodReturnValueHandler 处理返回值
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}

	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		// 对异步返回值的支持
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		// 遍历 HandlerMethodReturnValueHandler 数组，逐个判断是否支持
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			// 如果支持，则返回
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
```



## RequestResponseBodyMethodProcessor

org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor

继承 AbstractMessageConverterMethodProcessor 抽象类，处理添加了 @RequestBody 注解的入参，或添加了 @ResponseBody 注解的返回值。

前后端分离后，后端提供 Restful API，RequestResponseBodyMethodProcessor 成为了目前最常用的 HandlerMethodReturnValueHandler 实现类。此外它还是 HandlerMethodArgumentResolver 的实现类，拥有解析 @RequestBody 注解标注的请求入参的能力。

![HandlerMethodReturnValueHandler](https://image-hosting.jellyfishmix.com/20230220220156.png)



```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
	public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters) {
		super(converters);
	}

	public RequestResponseBodyMethodProcessor(List<HttpMessageConverter<?>> converters,
			@Nullable List<Object> requestResponseBodyAdvice) {

		super(converters, null, requestResponseBodyAdvice);
	}
}
```

converters 参数，HttpMessageConverter 数组，承载的元素 HttpMessageConverter，作用是将返回结果设置到响应中，供请求方获取。例如我们想要将 VO 对象，转换成 JSON 返回给请求方，就会用到 MappingJackson2HttpMessageConverter。

requestResponseBodyAdvice 参数，在父类 AbstractMessageConverterMethodArgumentResolver 中会将其转换成 RequestResponseBodyAdviceChain 对象 advice。

在 RequestMappingHandlerAdapter#afterPropertiesSet 方法中，第一步会初始化所有 ControllerAdvice 相关的类，然后在 getDefaultReturnValueHandlers 方法中，创建 RequestResponseBodyMethodProcessor 处理器时，会传入 requestResponseBodyAdvice 参数。

### 判断是否支持入参/返回值

org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#supportsParameter

org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#supportsReturnType

对于入参判断是否有 @RequestBody 注解，对于返回值判断是否有 @ResponseBody 注解。

```java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}

	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
				returnType.hasMethodAnnotation(ResponseBody.class));
	}
```

### resolveArgument -- 解析入参

org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#readWithMessageConverters

实现 HandlerMethodArgumentResolver 接口声明的方法，用于解析请求入参。

```java
	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		parameter = parameter.nestedIfOptional();
		// 从请求体中解析出方法入参值
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
		String name = Conventions.getVariableNameForParameter(parameter);

		// 数据绑定相关
		if (binderFactory != null) {
            // ...
		}

		// 返回方法入参对象，如果有必要，则通过 optional 获取对应的方法入参
		return adaptArgumentIfNecessary(arg, parameter);
	}
```

### AbstractMessageConverterMethodArgumentResolver#readWithMessageConverters 方法 -- 使用 HttpMessageConverter 读取 httpMessage 中的内容

org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodArgumentResolver#readWithMessageConverters

1. 从请求头中获取 Content-Type，为空则默认为 application/octet-stream
2. 获取方法参数的 contextClazz 和目标 clazz，用于 HttpMessageConverter 解析。
3. 获取 httpMethod，从请求中解析方法入参。
   1. 将 httpMessage 封装成 EmptyBodyCheckingHttpInputMessage，用于校验是否有请求体，没有的话设置为 null。
   2. 遍历 HttpMessageConverter 找到支持解析当前请求体的，从请求体中解析出方法入参对象。
   3. 如果请求体为空，则无需解析请求体。调用 RequestResponseBodyAdvice#afterBodyRead 方法，存在 RequestBodyAdvice 则对方法入参进行修改。
4. 异常处理和日志等收尾逻辑。

```java
	@Nullable
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
		// 获取使用的 MediaType，命名为 contentType
		MediaType contentType;
		boolean noContentType = false;
		// 从请求头中获取 Content-Type
		try {
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(ex.getMessage());
		}
		// contentType 为空则默认为 application/octet-stream
		if (contentType == null) {
			noContentType = true;
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}

		// 获取方法参数的 contextClazz 和目标 clazz，用于 HttpMessageConverter 解析
		Class<?> contextClass = parameter.getContainingClass();
		Class<T> targetClass = (targetType instanceof Class ? (Class<T>) targetType : null);
		if (targetClass == null) {
			ResolvableType resolvableType = ResolvableType.forMethodParameter(parameter);
			targetClass = (Class<T>) resolvableType.resolve();
		}

		// 获取 httpMethod
		HttpMethod httpMethod = (inputMessage instanceof HttpRequest ? ((HttpRequest) inputMessage).getMethod() : null);
		Object body = NO_VALUE;

		// 从请求中解析方法入参
		EmptyBodyCheckingHttpInputMessage message;
		try {
			// 将 httpMessage 封装成 EmptyBodyCheckingHttpInputMessage，用于校验是否有请求体，没有的话设置为 null
			message = new EmptyBodyCheckingHttpInputMessage(inputMessage);
			// 遍历 HttpMessageConverter
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				GenericHttpMessageConverter<?> genericConverter =
						(converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				// 如果该 HttpMessageConverter 能够读取当前请求体解析出方法入参
				if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
						(targetClass != null && converter.canRead(targetClass, contentType))) {
					// 如果请求体不为空
					if (message.hasBody()) {
						HttpInputMessage msgToUse =
								getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
						// 通过该 HttpMessageConverter 从请求体中解析出方法入参对象
						body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
								((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
						body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
					}
					else {
						// 如果请求体为空，则无需解析请求体。调用 RequestResponseBodyAdvice#afterBodyRead 方法，存在 RequestBodyAdvice 则对方法入参进行修改。
						body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
					}
					break;
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("I/O error while reading input message", ex, inputMessage);
		}

		// 校验解析出来的方法入参对象是否为空
		if (body == NO_VALUE) {
			if (httpMethod == null || !SUPPORTED_METHODS.contains(httpMethod) ||
					(noContentType && !message.hasBody())) {
				return null;
			}
			throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
		}

		// 打印日志...

		// 返回方法入参对象
		return body;
	}
```

### RequestResponseBodyMethodProcessor#handleReturnValue -- 处理返回值

org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor#handleReturnValue

处理返回值。

1. ModelAndViewContainer.requestHandled 属性设置为已处理，这样后续获取到的 ModelAndView 对象为 null。
2. 将 javax.servlet.http.HttpServletRequest/HttpServletResponse 封装成 org.springframework.http.server.ServletServerHttpRequest/ServletServerHttpResponse，便于操作。
3. 使用 HttpMessageConverter 对 returnValue 进行转换，并写入到 response。

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		// ModelAndViewContainer.requestHandled 属性设置为已处理，这样后续获取到的 ModelAndView 对象为 null
		mavContainer.setRequestHandled(true);
		// 创建 ServletServerHttpRequest 和 ServletServerHttpResponse
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		// 使用 HttpMessageConverter 对 returnValue 进行转换，并写入到 response
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
```

### AbstractMessageConverterMethodProcessor#writeWithMessageConverters -- 使用 HttpMessageConverter 对 returnValue 进行转换，并写入到 response

org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor#writeWithMessageConverters

使用 HttpMessageConverter 对 returnValue 进行转换，并写入到 response。

1. 如果是字符串则直接赋值。
2. 获取返回结果的类型(返回值 body 不为空则直接获取其类型，否则从返回结果类型 returnType 获取其返回值类型)。
3. Resource 类型的处理。
   1. 设置响应头 Accept-Ranges，数据不为空，请求头中的 Range 不为空，status 为 200。
   2. 断点续传，客户端已下载一部分数据，此时需要设置响应码为 206，获取哪一段数据需返回。
4. 选择使用的 MediaType，如果 response 中存在 ContentType 值，并且不包含通配符，则使用它作为 selectedMediaType。
5. 无法直接获取到 response 中的 ContentType 时的处理。
6. 获取到 selectedMediaType，进行写入逻辑。
   1. 移除 quality，例如 application/json;q=0.8 移除后为 application/json
   2. 遍历 messageConverters 数组，获取支持转换目标类型的 messageConverter
   3. 如果有 RequestResponseBodyAdvice，则需要对 returnValue 做切入。
   4. 核心逻辑，body(returnValue) 非空，使用 HttpMessageConverter 进行写入，returnValue 转换已完成, return
7. 如果上方没有 return 到达方法末尾，并且 body 非空，说明没有匹配的 HttpMessageConverter 转换器，抛出 HttpMediaTypeNotAcceptableException 异常。

```java
	@SuppressWarnings({"rawtypes", "unchecked"})
	protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		// body 来自于 returnValue
		Object body;
		// returnValueType
		Class<?> valueType;
		// 转换目标
		Type targetType;

		// 如果是字符串则直接赋值
		if (value instanceof CharSequence) {
			body = value.toString();
			valueType = String.class;
			targetType = String.class;
		}
		else {
			body = value;
			// 获取返回结果的类型(返回值 body 不为空则直接获取其类型，否则从返回结果类型 returnType 获取其返回值类型)
			valueType = getReturnValueType(body, returnType);
			targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
		}

		// Resource 类型的处理
		if (isResourceType(value, returnType)) {
			// 设置响应头 Accept-Ranges
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			// 数据不为空，请求头中的 Range 不为空，status 为 200
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					// 断点续传，客户端已下载一部分数据，此时需要设置响应码为 206
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					// 获取哪一段数据需返回
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}

		// 选择使用的 MediaType
		MediaType selectedMediaType = null;
		// 如果 response 中存在 ContentType 值，并且不包含通配符，则使用它作为 selectedMediaType
		MediaType contentType = outputMessage.getHeaders().getContentType();
		boolean isContentTypePreset = contentType != null && contentType.isConcrete();
		if (isContentTypePreset) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found 'Content-Type:" + contentType + "' in response");
			}
			selectedMediaType = contentType;
		}
		// 无法直接获取到 response 中的 ContentType 时的处理
		else {
            // ...
		}

		// 获取到 selectedMediaType，进行写入逻辑
		if (selectedMediaType != null) {
			// 移除 quality，例如 application/json;q=0.8 移除后为 application/json
			selectedMediaType = selectedMediaType.removeQualityValue();
			// 遍历 messageConverters 数组，获取支持转换目标类型的 messageConverter
			for (HttpMessageConverter<?> converter : this.messageConverters) {
				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ?
						(GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ?
						((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {
					// 如果有 RequestResponseBodyAdvice，则需要对 returnValue 做切入
					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					// 核心逻辑，body(returnValue) 非空进行写入
					if (body != null) {
						Object theBody = body;
						LogFormatUtils.traceDebug(logger, traceOn ->
								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");
						// 为 response Header 添加 CONTENT_DISPOSITION，一般情况下用不到
						addContentDispositionHeader(inputMessage, outputMessage);
						// 使用 HttpMessageConverter 写入内容
						if (genericConverter != null) {
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
						}
					}
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Nothing to write: null body");
						}
					}
					// returnValue 转换已完成, return
					return;
				}
			}
		}

		// 如果到达此处，并且 body 非空，说明没有匹配的 HttpMessageConverter 转换器，抛出 HttpMediaTypeNotAcceptableException 异常
		if (body != null) {
			Set<MediaType> producibleMediaTypes =
					(Set<MediaType>) inputMessage.getServletRequest()
							.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

			if (isContentTypePreset || !CollectionUtils.isEmpty(producibleMediaTypes)) {
				throw new HttpMessageNotWritableException(
						"No converter for [" + valueType + "] with preset Content-Type '" + contentType + "'");
			}
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
```



## ViewNameMethodReturnValueHandler

org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler

实现 HandlerMethodReturnValueHandler 接口，处理视图名 returnValue。

### supportsReturnType 方法

org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler#supportsReturnType

判断是否支持当前类型的 returnValue。支持的 returnType: void, String

```java
	/**
	 * 判断是否支持当前类型的 returnValue
	 */
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		Class<?> paramType = returnType.getParameterType();
		// 支持的 returnType: void, String
		return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
	}
```

### ViewNameMethodReturnValueHandler#handleReturnValue 方法

org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler#handleReturnValue

处理 returnValue。设置视图名到 mavContainer 中，如果是重定向，则标记到 mavContainer 中。

注意: 此时 mavContainer 的 requestHandled 属性，并未并未像 RequestResponseBodyMethodProcessor 一样，设置为 true 表示请求已处理。因为返回结果是视图名的场景下，需要使用 ViewResolver 从 ModelAndView 对象中解析出其对应的 view 对象，然后执行 View#render 方法，进行渲染。如果设置为 true，RequestMappingHandlerAdapter#getModelAndView 方法中获取到的 ModelAndView 对象就为 null 了，无法渲染视图。

```java
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		// 处理字符串类型入参
		if (returnValue instanceof CharSequence) {
			// 设置视图名到 mavContainer 中
			String viewName = returnValue.toString();
			mavContainer.setViewName(viewName);
			// 如果是重定向，则标记到 mavContainer 中
			if (isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 如果 returnType 是非 String 类型，而且非 void，则抛出 UnsupportedOperationException 异常
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}
```

