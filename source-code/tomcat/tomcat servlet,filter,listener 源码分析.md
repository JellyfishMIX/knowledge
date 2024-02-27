# tomcat servlet,filter,listener 源码分析



## 说明

1. 本文基于 tomcat 8.5.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## filter

### filter 调用链路

#### StandardWrapperValve#invoke 方法

org.apache.catalina.core.StandardWrapperValve#invoke

为请求分配 servlet，

1. 初始化一些局部变量，检查当前应用的状态是否可用等前置校验。
2. 调用 servlet 的 allocate 方法，为请求分配一个 servlet 实例。
3. 为此请求创建 ApplicationFilterChain 并调用 doFilter 方法，在 filterChain 中会调用 servlet 的 service 方法处理请求。

```java
    @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {
		// Initialize local variables we may need
        // 初始化一些局部变量
        boolean unavailable = false;
        Throwable throwable = null;
        // This should be a Request attribute...
        long t1=System.currentTimeMillis();
        requestCount.incrementAndGet();
        StandardWrapper wrapper = (StandardWrapper) getContainer();
        Servlet servlet = null;
        Context context = (Context) wrapper.getParent();

        // Check for the application being marked unavailable
        // 检查当前应用的状态是否可用
        if (!context.getState().isAvailable()) {
            response.sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                           sm.getString("standardContext.isUnavailable"));
            unavailable = true;
        }

		// 调用 servlet 的 allocate 方法，为请求分配一个 servlet 实例
        try {
            if (!unavailable) {
                servlet = wrapper.allocate();
            }
        } catch

		// Create the filter chain for this request
        // 为此请求创建 ApplicationFilterChain
        ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

        // Call the filter chain for this request
        // NOTE: This also calls the servlet's service() method
        // 为此请求调用 ApplicationFilterChain，在 filterChain 中会调用 servlet 的 service 方法处理请求
        Container container = this.container;
        try {
            if ((servlet != null) && (filterChain != null)) {
                // Swallow output if needed
                if (context.getSwallowOutput()) {
                    // ...
                } else {
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        // 核心方法
                        filterChain.doFilter
                            (request.getRequest(), response.getResponse());
                    }
                }
            }
        } catch () {
            // ...
        } finally {
            // ...
        }
}
```



### FilterChain 接口和 Filter 接口

1. FilterChain 接口和 Filter 接口应用了责任链设计模式，FilterChain 持有各 filter 的引用并排了序号，提供依次对各 filter 的调用能力。
2. 由 filterChain 负责从第一个 filter 开始，按序号调用各 filter 的 doFilter 方法。当前 filter 的 doFilter 方法执行完毕后，触发 filterChain 对下一个 filter 调用。所有 filter 调用完毕后，filterChain 负责调用 servlet 的 service 方法处理请求。

```java
public interface FilterChain {

    /**
     * Causes the next filter in the chain to be invoked, or if the calling
     * filter is the last filter in the chain, causes the resource at the end of
     * the chain to be invoked.
     *
     * @param request
     *            the request to pass along the chain.
     * @param response
     *            the response to pass along the chain.
     *
     * @throws IOException if an I/O error occurs during the processing of the
     *                     request
     * @throws ServletException if the processing fails for any other reason
     */
    public void doFilter(ServletRequest request, ServletResponse response)
            throws IOException, ServletException;

}
```

Filter 接口，提供对请求的过滤能力。声明了初始化方法 init 和销毁方法 destroy，核心是 doFilter 方法，子类实现后写过滤逻辑。

```java
public interface Filter {
    public void init(FilterConfig filterConfig) throws ServletException;

    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;

    public void destroy();
}
```

#### ApplicationFilterChain#doFilter 方法

filterChain 调用 filter 的封装方法，有一层校验逻辑的封装。直接看 internalDoFilter 方法，最终调用了 filter。

```java
public final class ApplicationFilterChain implements FilterChain {
	@Override
    public void doFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {

        // ...
        internalDoFilter(request,response);
}
```

#### ApplicationFilterChain#internalDoFilter 方法

filterChain 调用 filter 的具体方法。

1. n 是此 filterChain 上 filter 的数量，pos 是当前执行到的 filter 的序号。
2. 没执行完所有 filter，则继续调用下一个 filter 的 doFilter 方法处理请求。
3. 执行完所有 filter，到达 filterChain 的末尾，调用 servlet 实例的 service 方法处理请求。

```java
private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {
        // Call the next filter if there is one
        // n 是此 filterChain 上 filter 的数量，pos 是当前执行到的 filter 的序号
        // 没执行完所有 filter，则继续调用下一个 filter 的 doFilter 方法处理请求
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            try {
                Filter filter = filterConfig.getFilter();

                if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                        filterConfig.getFilterDef().getAsyncSupported())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
                }
                // ...
                filter.doFilter(request, response, this);
            } catch (IOException | ServletException | RuntimeException e) {
                // ...
            } catch (Throwable e) {
                // ...
            }
            return;
        }

        // We fell off the end of the chain -- call the servlet instance
        // 执行完所有 filter，到达 filterChain 的末尾，调用 servlet 实例的 service 方法处理请求
        try {
            // ...
            servlet.service(request, response);
        } catch (IOException | ServletException | RuntimeException e) {
            // ...
        } catch (Throwable e) {
            // ...
        } finally {
            // ...
        }
    }
```

### filter 的注册

#### spring boot 注册 filter demo

先看一段 spring boot 注册 filter 的 demo 代码，关键是拿到 ServletContext。

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class ServletWatcherInitializer extends SpringBootServletInitializer {

    /**
     * Logger
     */
    private static final Logger logger = LoggerFactory.getLogger(ServletWatcherInitializer.class);

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        logger.info("servlet_initializer_on_start_up");

        final ServletWatcher servletWatcher = ServletWatcher.withServletContext(servletContext);
        servletContext.addListener(servletWatcher);
        // 关键，使用 ServletContext 添加 filter
        servletContext.addFilter("servletWatcher", servletWatcher)
                .addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), false, "/*");
        super.onStartup(servletContext);
    }

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        logger.info("spring_application_configure");

        return application.sources(Application.class).applicationStartup(new ApplicationStartup());
    }
}
```

#### 使用 ServletContext 添加 filter

org.apache.catalina.core.ApplicationContext#addFilter(java.lang.String, java.lang.String, javax.servlet.Filter)

1. 添加一个 filter。
2. 注意这里只添加了只添加了 filterDefs。使用方还需要调用返回结果 FilterRegistration.Dynamic#addMappingForUrlPatterns 或 addMappingForServletNames 方法，把 filterMap 添加进 ServletContext。

```java
    /**
     * 添加一个 filter
     * 注意这里只添加了只添加了 filterDefs。
     * 使用方还需要调用返回结果 FilterRegistration.Dynamic#addMappingForUrlPatterns 或 addMappingForServletNames 方法，把 filterMap 添加进 ServletContext。
     */
    private FilterRegistration.Dynamic addFilter(String filterName,
            String filterClass, Filter filter) throws IllegalStateException {

        if (filterName == null || filterName.equals("")) {
            throw new IllegalArgumentException(sm.getString(
                    "applicationContext.invalidFilterName", filterName));
        }

        if (!context.getState().equals(LifecycleState.STARTING_PREP)) {
            //TODO Spec breaking enhancement to ignore this restriction
            throw new IllegalStateException(
                    sm.getString("applicationContext.addFilter.ise",
                            getContextPath()));
        }

        FilterDef filterDef = context.findFilterDef(filterName);

        // Assume a 'complete' FilterRegistration is one that has a class and
        // a name
        if (filterDef == null) {
            filterDef = new FilterDef();
            filterDef.setFilterName(filterName);
            // 添加一个 filterDef
            context.addFilterDef(filterDef);
        } else {
            if (filterDef.getFilterName() != null &&
                    filterDef.getFilterClass() != null) {
                return null;
            }
        }

        if (filter == null) {
            filterDef.setFilterClass(filterClass);
        } else {
            filterDef.setFilterClass(filter.getClass().getName());
            filterDef.setFilter(filter);
        }

        // 返回创建的 ApplicationFilterRegistration
        return new ApplicationFilterRegistration(filterDef, context);
    }
```

#### tomcat 创建 filterChain

org.apache.catalina.core.ApplicationFilterFactory#createFilterChain

1. 创建一个 ApplicationFilterChain。
2. 从 ServeltContext 中获取 filterMaps，遍历 filterMaps，通过 servletName 从 filterConfigs 中获取 ApplicationFilterConfig，向 filterChain 中添加 ApplicationFilterConfig。责任链中执行的 filter 载体就是 ApplicationFilterConfig。
3. 注意这里的细节，创建责任链时遍历的 filterMaps 不是 filterDefs，也就是说添加 filter 时只调用 ServletContext#addFilter 是不行的，只添加了 filterDefs。还需要调用 ServletContext#addFilter 的返回结果 FilterRegistration.Dynamic#addMappingForUrlPatterns 或 addMappingForServletNames 方法，把 filterMap 添加进 ServletContext。

```java
    public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {

        // If there is no servlet to execute, return null
        if (servlet == null) {
            return null;
        }

        // Create and initialize a filter chain object
        // 创建一个 ApplicationFilterChain
        ApplicationFilterChain filterChain = null;
        if (request instanceof Request) {
            Request req = (Request) request;
            if (Globals.IS_SECURITY_ENABLED) {
                // Security: Do not recycle
                filterChain = new ApplicationFilterChain();
            } else {
                filterChain = (ApplicationFilterChain) req.getFilterChain();
                if (filterChain == null) {
                    filterChain = new ApplicationFilterChain();
                    req.setFilterChain(filterChain);
                }
            }
        } else {
            // Request dispatcher in use
            filterChain = new ApplicationFilterChain();
        }

        filterChain.setServlet(servlet);
        filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

        // Acquire the filter mappings for this Context
        StandardContext context = (StandardContext) wrapper.getParent();
        // 获取 ServletContext 中的 filterMap
        FilterMap filterMaps[] = context.findFilterMaps();

        // If there are no filter mappings, we are done
        if ((filterMaps == null) || (filterMaps.length == 0)) {
            return filterChain;
        }

        // Acquire the information we will need to match filter mappings
        DispatcherType dispatcher =
                (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

        String requestPath = null;
        Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
        if (attribute != null){
            requestPath = attribute.toString();
        }

        String servletName = wrapper.getName();

        // Add the relevant path-mapped filters to this filter chain
        // 遍历 filterMap 向 filterChain 中添加 filter
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersURL(filterMap, requestPath)) {
                continue;
            }
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Add filters that match on servlet name second
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersServlet(filterMap, servletName)) {
                continue;
            }
            ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
                    context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                // FIXME - log configuration problem
                continue;
            }
            filterChain.addFilter(filterConfig);
        }

        // Return the completed filter chain
        return filterChain;
    }
```

#### StandardContext#filterStart

org.apache.catalina.core.StandardContext#filterStart

1. 把 filterDefs 转换成 filterConfigs，组织成便于通过 servletName 获取 ApplicationFilterConfig 的结构。

```java
    public boolean filterStart() {

        if (getLogger().isDebugEnabled()) {
            getLogger().debug("Starting filters");
        }
        // Instantiate and record a FilterConfig for each defined filter
        boolean ok = true;
        synchronized (filterConfigs) {
            filterConfigs.clear();
            for (Entry<String,FilterDef> entry : filterDefs.entrySet()) {
                String name = entry.getKey();
                if (getLogger().isDebugEnabled()) {
                    getLogger().debug(" Starting filter '" + name + "'");
                }
                try {
                    ApplicationFilterConfig filterConfig =
                            new ApplicationFilterConfig(this, entry.getValue());
                    filterConfigs.put(name, filterConfig);
                } catch (Throwable t) {
                    t = ExceptionUtils.unwrapInvocationTargetException(t);
                    ExceptionUtils.handleThrowable(t);
                    getLogger().error(sm.getString(
                            "standardContext.filterStart", name), t);
                    ok = false;
                }
            }
        }

        return ok;
    }
```

