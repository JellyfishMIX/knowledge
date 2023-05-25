# spring-web InvocableHandlerMethod 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

![HandlerMethod](https://image-hosting.jellyfishmix.com/20230203111227.png)

HandlerMethod，处理器的方法的封装对象。HandlerMethod 只提供了处理器的方法的基本信息，不提供调用逻辑。

InvocableHandlerMethod，继承 HandlerMethod 类，可调用的 HandlerMethod 实现类。



## ServletInvocableHandlerMethod#invokeAndHandle 方法

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#invokeAndHandle

核心方法，处理请求。

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



## ServletInvocableHandlerMethod#setResponseStatus 方法

org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod#setResponseStatus

调用 setResponseStatus(ServletWebRequest webRequest)  设置响应的状态码。

1. 获得 status，和 @ResponseStatus 注解相关。
2. 设置响应的状态码。有 reason，则设置 status + reason。无 reason，仅设置 status。
3. RedirectView 相关。

```java
	private void setResponseStatus(ServletWebRequest webRequest) throws IOException {
		// 获得 status，和 @ResponseStatus 注解相关
		HttpStatus status = getResponseStatus();
		if (status == null) {
			return;
		}

		// 设置响应的状态码
		HttpServletResponse response = webRequest.getResponse();
		if (response != null) {
			String reason = getResponseStatusReason();
			// 有 reason，则设置 status + reason
			if (StringUtils.hasText(reason)) {
				response.sendError(status.value(), reason);
			}
			else {
				// 无 reason，仅设置 status
				response.setStatus(status.value());
			}
		}

		// To be picked up by RedirectView
		// RedirectView 相关
		webRequest.getRequest().setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, status);
	}
```



## InvocableHandlerMethod

### InvocableHandlerMethod#invokeForRequest 方法 -- 执行调用处理请求

org.springframework.web.method.support.InvocableHandlerMethod#invokeForRequest

执行调用处理请求。

```java
	
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		// 解析参数
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Arguments: " + Arrays.toString(args));
		}
		// 执行调用
		return doInvoke(args);
	}
```

### InvocableHandlerMethod#getMethodArgumentValues 方法 -- 解析参数

org.springframework.web.method.support.InvocableHandlerMethod#getMethodArgumentValues

1. 获得方法的参数。
2. 遍历方法参数，将参数解析成对应的类型。
   1. 获得当前遍历的 MethodParameter 对象，给它设置 parameterNameDiscoverer。
   2. 从 providedArgs 中获得参数。如果获得到，则进入下一个参数的解析，默认情况 providedArgs 不会传参。
   3. 判断 resolvers 是否支持当前的参数解析。
   4. 执行参数解析，解析成功后进入下一个参数的解析。

```java
protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
		// 获得方法的参数
		MethodParameter[] parameters = getMethodParameters();
		// 无参，返回空数组
		if (ObjectUtils.isEmpty(parameters)) {
			return EMPTY_ARGS;
		}

		// 遍历方法参数，将参数解析成对应的类型
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			// 获得当前遍历的 MethodParameter 对象，给它设置 parameterNameDiscoverer
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			// 从 providedArgs 中获得参数。如果获得到，则进入下一个参数的解析，默认情况 providedArgs 不会传参。
			args[i] = findProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			// 判断 resolvers 是否支持当前的参数解析
			if (!this.resolvers.supportsParameter(parameter)) {
				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
			}
			try {
				// 执行参数解析，解析成功后进入下一个参数的解析
				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
			}
			catch (Exception ex) {
				// Leave stack trace for later, exception may actually be resolved and handled...
				if (logger.isDebugEnabled()) {
					String exMsg = ex.getMessage();
					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
						logger.debug(formatArgumentError(parameter, exMsg));
					}
				}
				throw ex;
			}
		}
		return args;
	}
```

### InvocableHandlerMethod#doInvoke 方法

org.springframework.web.method.support.InvocableHandlerMethod#doInvoke

1. 设置方法为可访问。
2. 通过反射执行方法调用。BridgedMethod 基于泛型安全为了和 jdk 1.5 之前的字节码兼容，无需关注 BridgedMethod。

```java
	@Nullable
	protected Object doInvoke(Object... args) throws Exception {
		// 设置方法为可访问
		ReflectionUtils.makeAccessible(getBridgedMethod());
		try {
			// 通过反射执行方法调用。BridgedMethod 基于泛型安全为了和 jdk 1.5 之前的字节码兼容，无需关注 BridgedMethod
			return getBridgedMethod().invoke(getBean(), args);
		}
		catch (IllegalArgumentException ex) {
			assertTargetBean(getBridgedMethod(), getBean(), args);
			String text = (ex.getMessage() != null ? ex.getMessage() : "Illegal argument");
			throw new IllegalStateException(formatInvokeError(text, args), ex);
		}
		catch (InvocationTargetException ex) {
			// Unwrap for HandlerExceptionResolvers ...
			Throwable targetException = ex.getTargetException();
			if (targetException instanceof RuntimeException) {
				throw (RuntimeException) targetException;
			}
			else if (targetException instanceof Error) {
				throw (Error) targetException;
			}
			else if (targetException instanceof Exception) {
				throw (Exception) targetException;
			}
			else {
				throw new IllegalStateException(formatInvokeError("Invocation failure", args), targetException);
			}
		}
	}
```



## 后言

1. 本文可以看到很重要的两个组件 HandlerMethodArgumentResolver 用于获取请求参数，HandlerMethodReturnValueHandler 用于处理返回值，后续分析。
