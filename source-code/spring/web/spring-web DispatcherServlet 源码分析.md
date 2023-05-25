# spring-web DispatcherServlet 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## DispatcherServlet 的继承实现层次

关注点应放在 Servlet, GenericServlet, HttpServlet, HttpServletBean, FrameworkServlet, DispatcherServlet

![image-20230109110424882](https://image-hosting.jellyfishmix.com/20230109110424.png)

### Servlet

javax.servlet.Servlet

```java
package javax.servlet;

public interface Servlet {

    public void init(ServletConfig config) throws ServletException;

    public ServletConfig getServletConfig();

    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;

    public String getServletInfo();

    public void destroy();
}
```

Servlet 接口提供处理请求的能力。请求来自于 web 容器，例如 tomcat, JBoss, Jetty 等。spring-web 应用和 web 容器共同遵守 Servlet 规范。Servlet 位于 javax.servlet 包内，不是 spring 的领域。

### GenericServlet

javax.servlet.GenericServlet

```java
package javax.servlet;

public abstract class GenericServlet implements Servlet, ServletConfig, java.io.Serializable {
	public void init(ServletConfig config) throws ServletException {
		this.config = config;
		this.init();
	}
}
```

GenericServlet 没有实现 service 方法，没有处理请求的逻辑。实现了 init 方法，但是个模板方法，由子类实现，例如 spring 的 HttpServletBean 就实现了该模板方法。GenericServlet 位于 javax.servlet 包内，不是 spring 的领域。

### HttpServlet

javax.servlet.http.HttpServlet

1. 基于 Servlet 规范的应用可以响应所有类型的请求(不仅仅是 HTTP，也可以是 WebSocket, FTP 等)，但通常用来处理 HTTP 请求。

2. Servlet 接口有一个非常重要的实现类 HttpServlet，实现了 service 方法，SpringMVC 框架中的 DispatcherServlet 就是继承自 HttpServlet。HttpServlet 位于 javax.servlet.http 包内，不是 spring 的领域。

```java
package javax.servlet.http;

public abstract class HttpServlet extends GenericServlet {
}
```

#### HttpServlet#service 方法

javax.servlet.http.HttpServlet#service(javax.servlet.ServletRequest, javax.servlet.ServletResponse)

1. 转换为 HttpServletRequest 和 HttpServletResponse。
2. 判断是何种类型的 http 请求，调用对应的 doXX 方法。GET 请求调用 doGet，POST 请求调用 doPost。doXX 方法是 HttpServlet 接口定义的。

```java
    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
        HttpServletRequest  request;
        HttpServletResponse response;

        try {
            request = (HttpServletRequest) req;
            response = (HttpServletResponse) res;
        } catch (ClassCastException e) {
            throw new ServletException("non-HTTP request or response");
        }
        service(request, response);
    }

    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod();
        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                if (ifModifiedSince < lastModified) {
                    // If the servlet mod time is later, call doGet()
                    // Round down to the nearest second for a proper compare
                    // A ifModifiedSince of -1 will always be less
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }
        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);
        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);
        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);
        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);
        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req,resp);
        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req,resp);
        } else {
            // Note that this means NO servlet supports whatever
            // method was requested, anywhere on this server.
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);
            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```

### FrameworkServlet

org.springframework.web.servlet.FrameworkServlet

1. FrameworkServlet 在 spring 的包中，进入到 spring 的领域。
2. FrameworkServlet 重写了 doGet(), doPost() 等 doXX 方法，doXX 方法调用了 processRequest() 方法。

```java
package org.springframework.web.servlet;

public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
}
```

#### FrameworkServlet#processRequest 方法

org.springframework.web.servlet.FrameworkServlet#processRequest

1. i18n 国际化设置。
2. 创建 ServletRequestAttributes 对象，初始化上下文 holders，将 request, response 对象放入到线程上下文中。
3. 调用 doService() 方法，处理请求。核心方法，由子类实现。

```java
	protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;
		// i18n 国际化设置
		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContext localeContext = buildLocaleContext(request);
		// 创建 ServletRequestAttributes 对象，初始化上下文 holders，将 request, response 对象放入到线程上下文中
		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
		initContextHolders(request, localeContext, requestAttributes);
        
		try {
			// 处理请求，核心方法，由子类实现
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
            // ...
		}
		catch (Throwable ex) {
            // ...
		} finally {
            // ...
		}
	}
```

#### FrameworkServlet#initServletBean 方法

org.springframework.web.servlet.FrameworkServlet#initServletBean

1. FrameworkServlet 还实现了 HttpServletBean 提供的模版方法 initServletBean，初始化 WebApplicationContext。调用 initFrameworkServlet 模板方法，供子类实现初始化工作。

```java
	protected final void initServletBean() throws ServletException {
        // 日志代码移除，保留核心代码
		// 初始化 WebApplicationContext
		this.webApplicationContext = initWebApplicationContext();
        // 模板方法，供子类实现初始化工作
		initFrameworkServlet();
	}
```



## DispatcherServlet#doService 方法

org.springframework.web.servlet.DispatcherServlet#doService

1. 首先判断是不是 include 请求，如果是则对 request 的 attribute 做个快照备份。
2. 为 request 设置 WebApplicationContext, 国际化 i18n 相关 resolver, theme 主题相关 resolver
3. 重定向属性处理，对 redirect 请求的支持。
4. 调用 DispatcherServlet#doDispatch 方法，核心方法，请求分发给对应的 handler 并处理。

```java
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		// 首先判断是不是 include 请求，如果是则对 request 的 attribute 做个快照备份
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		// 为 request 设置 WebApplicationContext
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		// 为 request 设置国际化 i18n 相关 resolver
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		// 为 reqeust 设置 theme 主题相关 resolver
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		// 重定向属性处理，对 redirect 请求的支持
		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		try {
			// 核心方法，请求分发给对应的 handler 并处理
			doDispatch(request, response);
		}
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				// doDispatch 方法执行完后，如果不是异步调用且未完成，对已备份好的快照进行还原，在做完快照后又对 request 设置了一些属性
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
```



## DispatcherServlet#doDispatch

org.springframework.web.servlet.DispatcherServlet#doDispatch

1. 先检查请求是不是 multipart 请求，如果是则返回已处理的 request，否则不对 request 做额外处理。
2. 遍历 HandlerMappings 集合，根据 HandlerMapping 获取 HandlerExecutionChain，即 mappedHandler。
3. 根据 mappedHandler 获取适配的 HandlerAdapter。
4. 以责任链模式调用 HandlerInterceptor 拦截器的 preHandle 方法(自定义的 spring 拦截器调用位置)。
5. 调用 HandlerAdapter#handle 方法，处理请求。实际 Controller 最终在这里面被调用，处理完后会返回 ModelAndView。
6. 将 HttpServletRequest 请求的 uri 转换成 ViewName 字符串并设置到 ModelAndView 中。
7. 以责任链模式调用 HandlerInterceptor 拦截器的 postHandle 方法(自定义的 spring 拦截器调用位置)。
8. 渲染 view 视图，调用拦截器的 afterCompletion() 方法。

```java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				// 先检查请求是不是 multipart 请求，如果是则返回已处理的 request，否则不对 request 做额外处理
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 遍历 HandlerMappings 集合，根据 HandlerMapping 获取 HandlerExecutionChain，即 mappedHandler
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				// 根据 mappedHandler 获取适配的 HandlerAdapter
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				// 以责任链模式调用 HandlerInterceptor 拦截器的 preHandle 方法(自定义的 spring 拦截器调用位置)
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 调用 HandlerAdapter#handle 方法，处理请求。实际 Controller 最终在这里面被调用，处理完后会返回 ModelAndView。
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				// 将 HttpServletRequest 请求的 uri 转换成 ViewName 字符串并设置到 ModelAndView 中
				applyDefaultViewName(processedRequest, mv);
				// 以责任链模式调用 HandlerInterceptor 拦截器的 postHandle 方法(自定义的 spring 拦截器调用位置)
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 渲染 view 视图，调用拦截器的 afterCompletion() 方法
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```



## DispatcherServlet#getHandler 方法

org.springframework.web.servlet.DispatcherServlet#getHandler

1. 遍历 handlerMapping 从中获得 HandlerExecutionChain
2. DispatcherServlet 初始化时注册的 handlerMapping, @see DispatcherServlet#initStrategies

```java
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		// DispatcherServlet 初始化时注册的 handlerMapping, @see DispatcherServlet#initStrategies
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```



## DispatcherServlet#getHandlerAdapter 方法

org.springframework.web.servlet.DispatcherServlet#getHandlerAdapter

根据 mappedHandler 获取适配的 HandlerAdapter。

1. 根据 HandlerExecutionChain 获取适配的 HandlerAdapter。责任链模式，遍历 HandlerAdapter 中调用 HandlerAdapter#support() 方法，判断当前 HandlerAdapter 是否支持该处理器，支持就返回该 HandlerAdapter。
2. DispatcherServlet 初始化时注册的 HandlerAdapter, @see DispatcherServlet#initStrategies

```java
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```



## DispatcherServlet#applyDefaultViewName 方法

org.springframework.web.servlet.DispatcherServlet#applyDefaultViewName

将 HttpServletRequest 请求的 uri 转换成 ViewName 字符串并设置到 ModelAndView 中

1. 判断 ModelAndView 是否需要 view 视图层。
2. getDefaultViewName 方法使用了 RequestToViewNameTranslator，将 HttpServletRequest 请求的 uri 转换成 ViewName 字符串并设置到 ModelAndView 中。

```java
	private void applyDefaultViewName(HttpServletRequest request, @Nullable ModelAndView mv) throws Exception {
		// 判断 ModelAndView 是否需要 view 视图层
        if (mv != null && !mv.hasView()) {
            // getDefaultViewName 方法使用了 RequestToViewNameTranslator，将 HttpServletRequest 请求的 uri 转换成 ViewName 字符串并设置到 ModelAndView 中
			String defaultViewName = getDefaultViewName(request);
			if (defaultViewName != null) {
				mv.setViewName(defaultViewName);
			}
		}
	}

	@Nullable
	protected String getDefaultViewName(HttpServletRequest request) throws Exception {
		return (this.viewNameTranslator != null ? this.viewNameTranslator.getViewName(request) : null);
	}

	@Override
	public String getViewName(HttpServletRequest request) {
		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, HandlerMapping.LOOKUP_PATH);
		return (this.prefix + transformPath(lookupPath) + this.suffix);
	}
```



## DispatcherServlet#processDispatchResult 方法 -- 处理 Handler 执行后的结果

org.springframework.web.servlet.DispatcherServlet#processDispatchResult

处理 Handler 执行后的结果，将 ModelAndView 或者 Exception 渲染成视图，然后调用 render() 生成页面。

1. handler 执行后如果存在 Exception，责任链模式调用 HandlerExceptionResolver，将 Exception 处理成 ModelAndView。
2. 如果 ModelAndView 不为空，调用 render 方法生成页面。
3. 以责任链模式调用 HandlerInterceptor#afterCompletion() 方法。

```java
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;
		// handler 执行后如果存在 Exception，将 Exception 处理为 ModelAndView
		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				// 责任链模式调用 HandlerExceptionResolver，将 Exception 处理成 ModelAndView
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
			// 如果 ModelAndView 不为空，调用 render 方法生成页面
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
        // ...

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		// 以责任链模式调用 HandlerInterceptor#afterCompletion() 方法
		if (mappedHandler != null) {
			// Exception (if any) is already handled..
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```



## DispatcherServlet#render 方法 -- 生成 View

org.springframework.web.servlet.DispatcherServlet#render

1. 责任链模式调用 ViewResolver#resolveViewName 方法，生成 View

```java
	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale = (this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);

		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			// 责任链模式调用 ViewResolver#resolveViewName 方法，生成 View
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		} else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view [" + view + "] ");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
			view.render(mv.getModelInternal(), request, response);
		} catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "]", ex);
			}
			throw ex;
		}
	}
```
