# spring-web HandlerMethodArgumentResolver 源码分析



## HandlerMethodArgumentResolver 的使用处，解析参数

org.springframework.web.method.support.InvocableHandlerMethod#getMethodArgumentValues

HandlerMethodArgumentResolver 在解析参数时使用，以下是使用处 InvocableHandlerMethod#getMethodArgumentValues 方法。

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



## 类层次

![img](https://image-hosting.jellyfishmix.com/20230209113328.png)



## HandlerMethodArgumentResolver 接口

org.springframework.web.method.support.HandlerMethodArgumentResolver

```java
public interface HandlerMethodArgumentResolver {
	/**
	 * 是否支持解析当前参数
	 */
	boolean supportsParameter(MethodParameter parameter);

	/**
	 * 解析当前参数
	 */
	@Nullable
	Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}
```

### HandlerMethodArgumentResolverComposite -- 复合体

org.springframework.web.method.support.HandlerMethodArgumentResolverComposite

1. 实现了 HandlerMethodArgumentResolver 接口，复合的 HandlerMethodArgumentResolver 实现类。使用了组合模式，作为一个整体承载了 argumentResolvers。

2. 获取 argumentResolver 时，如果没有缓存，则需要遍历 argumentResolvers 集合获取当前 methodParameter 适合的 argumentResolver。因此引入 argumentResolverCache，作为 MethodParameter 与 HandlerMethodArgumentResolver 的映射缓存，用于快速获取。

```java
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {
	/**
	 * 存储 argumentResolvers 的集合
	 */
	private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>();

	/**
	 * MethodParameter 与 HandlerMethodArgumentResolver 的映射，作为缓存用于快速获取
	 */
	private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache = new ConcurrentHashMap<>(256);
}
```

### HandlerMethodArgumentResolverComposite#resolveArgument 方法 -- 解析参数

org.springframework.web.method.support.HandlerMethodArgumentResolverComposite#resolveArgument

解析参数，应用了策略模式。

1. 获取当前 methodParameter 适合的 argumentResolver。
2. 使用 argumentResolver 解析参数。

```java
	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		// 获取当前 methodParameter 适合的 argumentResolver
		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		if (resolver == null) {
			throw new IllegalArgumentException("Unsupported parameter type [" +
					parameter.getParameterType().getName() + "]. supportsParameter should be called first.");
		}
		// 使用 argumentResolver 解析参数
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}
```

### HandlerMethodArgumentResolverComposite#getArgumentResolver 方法-- 获取当前 methodParameter 适合的 argumentResolver

org.springframework.web.method.support.HandlerMethodArgumentResolverComposite#getArgumentResolver

获取当前 methodParameter 适合的 argumentResolver。

1. 优先从 argumentResolverCache 缓存中，获取当前 methodParameter 适合的 argumentResolver。
2. 缓存中没有，遍历 argumentResolvers 集合获取当前 methodParameter 适合的 argumentResolver，HandlerMethodArgumentResolver#supportsParameter 方法用于判断是否支持。支持的 argumentResolver 加入缓存。

```java
	/**
	 * Find a registered {@link HandlerMethodArgumentResolver} that supports
	 * the given method parameter.
	 *
	 * 获取当前 methodParameter 适合的 argumentResolver
	 */
	@Nullable
	private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
		// 优先从 argumentResolverCache 缓存中，获取当前 methodParameter 适合的 argumentResolver
		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
		// 缓存中没有，遍历 argumentResolvers 集合获取当前 methodParameter 适合的 argumentResolver
		if (result == null) {
			for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
				// 是否支持
				if (resolver.supportsParameter(parameter)) {
					result = resolver;
                    // 支持的 argumentResolver 加入缓存，break 跳出遍历
					this.argumentResolverCache.put(parameter, result);
					break;
				}
			}
		}
		return result;
	}
```



## AbstractNamedValueMethodArgumentResolver

org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver

基于名称获取值的 HandlerMethodArgumentResolver 抽象基类，提供根据参数名称解析的能力。

例如，@RequestParam(value = "username") 注解，使用 AbstractNamedValueMethodArgumentResolver 从请求中获得 username 对应的参数值(更具体地说是其子类 RequestParamMethodArgumentResolver)。

子类主要有，RequestParamMethodArgumentResolver, RequestParamMapMethodArgumentResolver 和 PathVariableMethodArgumentResolver。 

```java
public abstract class AbstractNamedValueMethodArgumentResolver implements HandlerMethodArgumentResolver {
	/**
	 * MethodParameter 和 NamedValueInfo 的映射，作为缓存
	 */
	private final Map<MethodParameter, NamedValueInfo> namedValueInfoCache = new ConcurrentHashMap<>(256);
}
```

### NamedValueInfo

org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver.NamedValueInfo

NamedValueInfo 是静态内部类，对应注解的属性。

```java
	/**
	 * Represents the information about a named value, including name, whether it's required and a default value.
	 * 静态内部类，对应注解的属性
	 */
	protected static class NamedValueInfo {
		/**
		 * 名称
		 */
		private final String name;

		/**
		 * 是否必填
		 */
		private final boolean required;

		/**
		 * 默认值
		 */
		@Nullable
		private final String defaultValue;

		public NamedValueInfo(String name, boolean required, @Nullable String defaultValue) {
			this.name = name;
			this.required = required;
			this.defaultValue = defaultValue;
		}
	}
```

### AbstractNamedValueMethodArgumentResolver#resolveArgument 方法

org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver#resolveArgument

解析请求参数。

1. 获得方法参数对应的 namedValueInfo 对象。
2. 如果 parameter 是 Optional 类型，则获取内嵌的参数。否则使用 parameter 本身。
3. 如果 name 是 spel 表达式(占位符)，则解析成真实的 name。
4. 解析 name 对应的值 arg，由子类实现。如果 arg 为空，则使用默认值。默认值不存在则处理参数缺省和空值的情况。
5. 数据绑定相关。
6. 后置处理解析的参数值。

```java
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		// 获得方法参数对应的 namedValueInfo 对象
		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		// 如果 parameter 是 Optional 类型，则获取内嵌的参数。否则使用 parameter 本身
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		// 如果 name 是 spel 表达式(占位符)，则解析成真实的 name
		Object resolvedName = resolveEmbeddedValuesAndExpressions(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException(
					"Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}

		// 解析 name 对应的值，由子类实现
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		// 如果 arg 为空，则使用默认值。默认值不存在则处理参数缺省和空值的情况。
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
			}
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveEmbeddedValuesAndExpressions(namedValueInfo.defaultValue);
		}

		// 数据绑定相关
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			}
			catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
		}

		// 后置处理解析的参数值
		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

		return arg;
	}
```

### AbstractNamedValueMethodArgumentResolver#getNamedValueInfo 方法 -- 获取 namedValueInfo

org.springframework.web.method.annotation.AbstractNamedValueMethodArgumentResolver#getNamedValueInfo

获取 namedValueInfo，先尝试从缓存中获取，没有则创建并加入缓存。

```java
	/**
	 * Obtain the named value for the given method parameter.
	 */
	private NamedValueInfo getNamedValueInfo(MethodParameter parameter) {
		// 从 namedValueInfoCache 缓存中，获得 namedValueInfo 对象
		NamedValueInfo namedValueInfo = this.namedValueInfoCache.get(parameter);
		if (namedValueInfo == null) {
			// 获得不到 namedValueInfo 则创建，这是一个抽象方法，由子类具体实现
			namedValueInfo = createNamedValueInfo(parameter);
			// 更新 namedValueInfo 对象
			namedValueInfo = updateNamedValueInfo(parameter, namedValueInfo);
			// 添加到 namedValueInfoCache 缓存中
			this.namedValueInfoCache.put(parameter, namedValueInfo);
		}
		return namedValueInfo;
	}
```



## RequestParamMethodArgumentResolver

处理普通的请求参数。

```java
public class RequestParamMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver implements UriComponentsContributor {
}
```

### RequestParamMethodArgumentResolver#createNamedValueInfo 方法 -- 创建方法参数对应的 namedValueInfo 对象

org.springframework.web.method.annotation.RequestParamMethodArgumentResolver#createNamedValueInfo

1. 如果方法参数有 @RequestParam 注解，则根据注解创建一个 RequestParamNamedValueInfo 对象，获取注解中的 name, required 和 defaultValue 配置。
2. 否则，就创建一个空的 RequestParamNamedValueInfo 对象，三个属性分别为，空字符串"", false 和 ValueConstants.DEFAULT_NONE。

```java
	@Override
	protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
		RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
		return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
	}

	private static class RequestParamNamedValueInfo extends NamedValueInfo {

		public RequestParamNamedValueInfo() {
			super("", false, ValueConstants.DEFAULT_NONE);
		}

		public RequestParamNamedValueInfo(RequestParam annotation) {
			super(annotation.name(), annotation.required(), annotation.defaultValue());
		}
	}
```

### RequestParamMethodArgumentResolver#supportsParameter -- 是否支持解析当前参数

org.springframework.web.method.annotation.RequestParamMethodArgumentResolver#supportsParameter

判断此 argumentResolver 是否支持解析当前参数。

1. 有 @RequestParam 注解的情况。
   1. 如果是 Map 类型，则 @RequestParam 注解必须要有 name 属性。
   2. 有 @RequestParam 注解的情况且非 Map，则直接适用。
2. 没有 @RequestParam 注解的情况。
   1. 如果有 @RequestPart 注解，返回 false。
   2. 获得参数，内嵌(Optional)参数则获得内嵌值，非内嵌参数直接用 parameter 本身。
   3. 如果是 Multipart 参数，则适用。如果开启 useDefaultResolution 功能，则判断是否为普通类型。

```java
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		// 有 @RequestParam 注解的情况
		if (parameter.hasParameterAnnotation(RequestParam.class)) {
			// 如果是 Map 类型，则 @RequestParam 注解必须要有 name 属性
			if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
				RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
				return (requestParam != null && StringUtils.hasText(requestParam.name()));
			}
			// 有 @RequestParam 注解的情况且非 Map，则直接适用
			else {
				return true;
			}
		}
		// 没有 @RequestParam 注解的情况
		else {
			// 如果有 @RequestPart 注解，返回 false
			if (parameter.hasParameterAnnotation(RequestPart.class)) {
				return false;
			}
			// 获得参数，内嵌(Optional)参数则获得内嵌值，非内嵌参数直接用 parameter 本身
			parameter = parameter.nestedIfOptional();
			// 如果是 Multipart 参数，则适用
			if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
				return true;
			}
			// 如果开启 useDefaultResolution 功能，则判断是否为普通类型
			else if (this.useDefaultResolution) {
				return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
			}
			else {
				return false;
			}
		}
	}
```

### RequestParamMethodArgumentResolver#resolveName

org.springframework.web.method.annotation.RequestParamMethodArgumentResolver#resolveName

解析参数 name 对应的值。

1. 情况1，HttpServletRequest 情况下的 MultipartFile 和 Part 的情况。
2. 情况2，MultipartHttpServletRequest 情况下的 MultipartFile 的情况。
3. 情况3，普通参数的获取。例如常见的 String, Integer 之类的请求参数，直接从请求中获取参数值。

情况1和2，处理参数类型为文件(org.springframework.web.multipart.MultipartFile) 和 javax.servlet.http.Part 参数的获取。

```java
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		// 情况一，HttpServletRequest 情况下的 MultipartFile 和 Part
		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

		if (servletRequest != null) {
			Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
			if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
				return mpArg;
			}
		}

		Object arg = null;
		// 情况二，MultipartHttpServletRequest 情况下的 MultipartFile 的情况
		MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
		if (multipartRequest != null) {
			List<MultipartFile> files = multipartRequest.getFiles(name);
			if (!files.isEmpty()) {
				arg = (files.size() == 1 ? files.get(0) : files);
			}
		}

		// 情况三，普通参数的获取
		if (arg == null) {
			String[] paramValues = request.getParameterValues(name);
			if (paramValues != null) {
				arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
			}
		}
		return arg;
	}
```



## RequestParamMapMethodArgumentResolver

org.springframework.web.method.annotation.RequestParamMapMethodArgumentResolver

实现了 HandlerMethodArgumentResolver 接口，用于处理带有 @RequestParam 注解，但是注解上没有 name 属性的 Map 类型的参数。

```java
public class RequestParamMapMethodArgumentResolver implements HandlerMethodArgumentResolver {
}
```

### RequestParamMapMethodArgumentResolver 和 RequestParamMethodArgumentResolver 对比

RequestParamMapMethodArgumentResolver 的机制是，将所有参数添加到 Map 集合中

```java
@RequestMapping("/hello")
public String hello4(@RequestParam Map<String, Object> map) {
    return "666";
}
```

发送请求 `GET /hello?name=yyy&age=20`，name 和 age 参数，就会都添加到 map 中。

RequestParamMethodArgumentResolver 的机制是，将指定名字的参数添加到 Map 集合中。

```java
@RequestMapping("/hello")
public String hello5(@RequestParam(name = "map") Map<String, Object> map) {
    return "666";
}
```

发送请求 GET /hello4?map={"name": "yyyy", age: 20}， map 参数的元素都会添加到方法参数 map 中。当然，要注意下，实际请求要 UrlEncode 编码下参数，实际请求是 `GET /hello?map=%7b%22name%22%3a+%22yyyy%22%2c+age%3a+20%7d`



## PathVariableMethodArgumentResolver

org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver

实现 UriComponentsContributor 接口，继承 AbstractNamedValueMethodArgumentResolver 抽象类，从请求路径中解析参数。

```java
public class PathVariableMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver implements UriComponentsContributor {
}
```

### supportsParameter

org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver#supportsParameter

判断此 argumentResolver 是否支持解析当前参数。

```java
	/**
	 * 判断此 argumentResolver 是否支持解析当前参数
	 */
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		// 必须有 @PathVariable 注解才能支持
		if (!parameter.hasParameterAnnotation(PathVariable.class)) {
			return false;
		}
		// 请求参数如果是 Map 类型，必须有 @PathVariable 注解且有 name 属性才能支持
		if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
			PathVariable pathVariable = parameter.getParameterAnnotation(PathVariable.class);
			return (pathVariable != null && StringUtils.hasText(pathVariable.value()));
		}
		return true;
	}
```

### resolveName

org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver#resolveName

实现 resolveName(String name, MethodParameter parameter, NativeWebRequest request) 方法，从请求路径中获取方法参数的值。

```java
	@Override
	@SuppressWarnings("unchecked")
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		// 获取请求路径参数
		Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(
				HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
		// 获取参数值
		return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
	}
```

