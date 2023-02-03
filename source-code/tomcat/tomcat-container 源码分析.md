# tomcat-container 源码分析



## 说明

1. 本文基于 tomcat 8.5.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## tomcat 的 container 容器

1. tomcat 由 connector 和 container 两部分组成，connector 接收到请求后，先将请求包装为 request，然后传递给 container 进行处理，最终返回给请求方。
2. tomcat-container 按照包含关系一共有 4 个容器: StandardEngine, StandardHost, StandardContext, StandardWrapper。 
3. wrapper 表示一个 servlet，context 表示一个 web 应用程序，一个 web 应用程序中可能有多个 servlet。host 表示一个虚拟主机，一个虚拟主机里可能运行着多个 web 应用程序。engine 控制一个虚拟主机的生命周期。

![img](https://image-hosting.jellyfishmix.com/20221227165713.png)



## tomcat 的 pipeline 管道和 valve 阀门

1. container 模块中，4 个 container 间不会直接调用对方来传递请求，请求在 4 个 container 之间传递依靠 pipeline 管道，调用 pipeline 传递请求。
2. 4 种容器都有自己的 pipeline 组件，每个 pipeline 组件上可以有多个 valve(阀门)，且至少会设定 1 个 baseValve(基础阀门)。基础阀门的作用是连接下一个 container，baseValve 是两个 container 之间的桥梁。pipeline-valve 应用了设计模式--责任链模式。
3. pipeline 定义对应的接口是 Pipeline, 具体实现是 StandardPipeline。valve 定义对应的接口是 Valve，抽象类是 ValveBase。4 个容器对应基础阀门分别是 StandardEngineValve, StandardHostValve, StandardContextValve, StandardWrapperValve。



## Pipeline 接口

org.apache.catalina.Pipeline

具体实现是 StandardPipeline

1. Pipeline 中很多的方法用于操作 Valve，包括获取，设置，移除 Valve。
2. getFirst 方法返回的是 Pipeline 上的第一个 Valve，addValve 方法可以添加一个阀门。getBasic/setBasic 方法用于获取/设置基础阀。basicValve 位于 pipeline 最后一个阀门位置。

```java
public interface Pipeline {
    public Valve getBasic();

    public void setBasic(Valve valve);

    public void addValve(Valve valve);

    public Valve[] getValves();

    public void removeValve(Valve valve);

    public Valve getFirst();

    public boolean isAsyncSupported();

    public Container getContainer();

    public void setContainer(Container container);
}
```

### StandardPipeline#startInternal 方法

org.apache.catalina.core.StandardPipeline#startInternal

启动此 pipeline

1. 设置当前 valve，如果 first 引用指向了 valve，则 first 指向的 valve 为当前 valve。否则 basicValve 为当前 valve。
2. 遍历此 pipeline 的 valve 链表，调用链上每个 valve 的 start 方法。
3. 将 pipeline 状态置为 STARTING

```java
    @Override
    protected synchronized void startInternal() throws LifecycleException {
        // Start the Valves in our pipeline (including the basic), if any
        // 设置当前 valve，如果 first 引用指向了 valve，则 first 指向的 valve 为当前 valve。否则 basicValve 为当前 valve。
        Valve current = first;
        if (current == null) {
            current = basic;
        }
        // 遍历此 pipeline 的 valve 链表，调用链上每个 valve 的 start 方法
        while (current != null) {
            if (current instanceof Lifecycle) {
                ((Lifecycle) current).start();
            }
            current = current.getNext();
        }
        // 将 pipeline 状态置为 STARTING
        setState(LifecycleState.STARTING);
    }
```

### StandardPipeline#setBasic 方法

org.apache.catalina.core.StandardPipeline#setBasic

为此 pipeline 设置基础阀门 basicValve

1. 如果已经有 basicValve 并且和要设置的值一样，那么直接 return 结束设置。
2. 如果旧的 basicValve 非空，则调用其 stop 方法，取消和对应 container 的关联。
3. 启动新的 basicValve 并和 container 进行关联。
4. 遍历阀门链表，将新的 basicValve 替换至阀门链表。

```java
    @Override
    public void setBasic(Valve valve) {

        // Change components if necessary
        // 如果已经有 basicValve 并且和要设置的值一样，那么直接 return 结束设置
        Valve oldBasic = this.basic;
        if (oldBasic == valve) {
            return;
        }

        // Stop the old component if necessary
        // 如果旧的 basicValve 非空，则调用其 stop 方法，取消和对应 container 的关联
        if (oldBasic != null) {
            if (getState().isAvailable() && (oldBasic instanceof Lifecycle)) {
                try {
                    ((Lifecycle) oldBasic).stop();
                } catch (LifecycleException e) {
                    log.error(sm.getString("standardPipeline.basic.stop"), e);
                }
            }
            if (oldBasic instanceof Contained) {
                try {
                    ((Contained) oldBasic).setContainer(null);
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                }
            }
        }

        // Start the new component if necessary
        if (valve == null) {
            return;
        }
        // 新的 basicValve 和 container 进行关联
        if (valve instanceof Contained) {
            ((Contained) valve).setContainer(this.container);
        }
        // 启动新的 basicValve
        if (getState().isAvailable() && valve instanceof Lifecycle) {
            try {
                ((Lifecycle) valve).start();
            } catch (LifecycleException e) {
                log.error(sm.getString("standardPipeline.basic.start"), e);
                return;
            }
        }

        // Update the pipeline
        // 遍历阀门链表，将新的 basicValve 替换至阀门链表
        Valve current = first;
        while (current != null) {
            if (current.getNext() == oldBasic) {
                current.setNext(valve);
                break;
            }
            current = current.getNext();
        }

        this.basic = valve;

    }
```

### StandardPipeline#addValve 方法

org.apache.catalina.core.StandardPipeline#addValve

为此 pipeline 添加 valve

1. 启动 valve 并和 container 进行关联。
2. first 指向阀门链表第一个元素，从 first 开始遍历阀门链表，把 valve 加在现有 valve 链表末尾，basicValve 之前。
3. 触发 container 添加 valve 事件。

```java
	public void addValve(Valve valve) {

        // Validate that we can add this Valve
        // valve 和 container 进行关联
        if (valve instanceof Contained) {
            ((Contained) valve).setContainer(this.container);
        }

        // Start the new component if necessary
        // 启动 valve
        if (getState().isAvailable()) {
            if (valve instanceof Lifecycle) {
                try {
                    ((Lifecycle) valve).start();
                } catch (LifecycleException e) {
                    log.error(sm.getString("standardPipeline.valve.start"), e);
                }
            }
        }

        // Add this Valve to the set associated with this Pipeline
        // first 指向阀门链表第一个元素，从 first 开始遍历阀门链表，把 valve 加在现有 valve 链表末尾，basicValve 之前
        if (first == null) {
            first = valve;
            valve.setNext(basic);
        } else {
            Valve current = first;
            while (current != null) {
                if (current.getNext() == basic) {
                    current.setNext(valve);
                    valve.setNext(basic);
                    break;
                }
                current = current.getNext();
            }
        }
        // 触发 container 添加 valve 事件
        container.fireContainerEvent(Container.ADD_VALVE_EVENT, valve);
    }
```

### StandardPipeline#getValves 方法

org.apache.catalina.core.StandardPipeline#getValves

获取此 pipeline 所有的 valve，数组形式返回。

1. 遍历链表把 valve 添加进集合内，最后集合转数组。

```java
    /**
     * 获取此 pipeline 所有的 valve，数组形式返回。
     */
    @Override
    public Valve[] getValves() {
        // 遍历链表把 valve 添加进集合内，最后集合转数组
        List<Valve> valveList = new ArrayList<>();
        Valve current = first;
        if (current == null) {
            current = basic;
        }
        while (current != null) {
            valveList.add(current);
            current = current.getNext();
        }

        return valveList.toArray(new Valve[0]);
    }
```

### StandardPipeline#removeValve 方法

org.apache.catalina.core.StandardPipeline#removeValve

移除指定的 valve

1. 遍历阀门链表，定位目标 valve 并移除。
2. first 是严格定义的除了 basicValve 的第一个阀门，不包括 basicValve。
3. 取消被移除的 valve 与 container 的关联。
4. 调用被移除 valve 的 stop, destroy 方法。
5. 触发 container 移除 valve 事件。

```java
    /**
     * Remove the specified Valve from the pipeline associated with this
     * Container, if it is found; otherwise, do nothing.  If the Valve is
     * found and removed, the Valve's <code>setContainer(null)</code> method
     * will be called if it implements <code>Contained</code>.
     *
     * 移除指定的 valve
     *
     * @param valve Valve to be removed
     */
    @Override
    public void removeValve(Valve valve) {

        Valve current;
        if(first == valve) {
            first = first.getNext();
            current = null;
        } else {
            current = first;
        }
        // 遍历阀门链表，定位目标 valve 并移除
        while (current != null) {
            if (current.getNext() == valve) {
                current.setNext(valve.getNext());
                break;
            }
            current = current.getNext();
        }
        // first 是严格定义的除了 basicValve 的第一个阀门，不包括 basicValve
        if (first == basic) {
            first = null;
        }

        // 取消被移除的 valve 与 container 的关联
        if (valve instanceof Contained) {
            ((Contained) valve).setContainer(null);
        }

        // 调用被移除 valve 的 stop, destroy 方法
        if (valve instanceof Lifecycle) {
            // Stop this valve if necessary
            if (getState().isAvailable()) {
                try {
                    ((Lifecycle) valve).stop();
                } catch (LifecycleException e) {
                    log.error(sm.getString("standardPipeline.valve.stop"), e);
                }
            }
            try {
                ((Lifecycle) valve).destroy();
            } catch (LifecycleException e) {
                log.error(sm.getString("standardPipeline.valve.destroy"), e);
            }
        }
        // 触发 container 移除 valve 事件。
        container.fireContainerEvent(Container.REMOVE_VALVE_EVENT, valve);
    }
```

### StandardPipeline#getFirst 方法

org.apache.catalina.core.StandardPipeline#getFirst

获取此 pipeline 第一个 valve，如果没有普通 valve 则返回 basicValve

```java
/**
 * 获取此 pipeline 第一个 valve，如果没有普通 valve 则返回 basicValve
 */
@Override
public Valve getFirst() {
    if (first != null) {
        return first;
    }

    return basic;
}
```



## Valve 接口

org.apache.catalina.Valve

一个 pipeline 可以有多个 valve，这些 valve 链式存储，上一个 valve 对象持有下一个 valve 对象的引用。调用 getNext() 方法即可获取下个 valve 实例。setNext 方法设置下一个 valve 实例。

```java
public interface Valve {
    public String getInfo();

    public Valve getNext();

    public void setNext(Valve valve);

    public void backgroundProcess();

    public void invoke(Request request, Response response) throws IOException, ServletException;

    public void event(Request request, Response response, CometEvent event) throws IOException,ServletException;
    
    public boolean isAsyncSupported();
}
```

### StandardEngineValve#invoke 方法

org.apache.catalina.core.StandardEngineValve#invoke

边界条件处理，然后调用 hostPipeline 的 valve 链表。

```java
    @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Select the Host to be used for this Request
        Host host = request.getHost();
        if (host == null) {
            // HTTP 0.9 or HTTP 1.0 request without a host when no default host
            // is defined.
            // Don't overwrite an existing error
            if (!response.isError()) {
                response.sendError(404);
            }
            return;
        }
        if (request.isAsyncSupported()) {
            request.setAsyncSupported(host.getPipeline().isAsyncSupported());
        }

        // Ask this Host to process this request
        host.getPipeline().getFirst().invoke(request, response);
    }
```

### StandardHostValve#invoke 方法

org.apache.catalina.core.StandardHostValve#invoke

边界条件处理，然后调用 contextPipeline 的 valve 链表。

```java
public final class StandardHostValve extends ValveBase {
    protected Container container = null;
    protected Valve next = null;
    
    @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {
        Context context = request.getContext();
        if (context == null) {
            return;
        }
        if (request.isAsyncSupported()) {
            request.setAsyncSupported(context.getPipeline()
                    .isAsyncSupported());
        }
        boolean asyncAtStart = request.isAsync();
        try {
            context.bind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
            if (!asyncAtStart && 
                    !context.fireRequestInitEvent(request.getRequest())) {
                return;
            }
            try {
                if (!response.isErrorReportRequired()) {
                    // 继续执行管道连接的下个Container方法
                    context.getPipeline().getFirst()
                            .invoke(request, response);
                }
            } catch (Throwable t) {
                ...
            }
            ...
        } finally {
            if (ACCESS_SESSION) {
                request.getSession(false);
            }
            context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
        }
    }
}
```

### StandardContextValve#invoke 方法

org.apache.catalina.core.StandardContextValve#invoke

禁止访问 WEB-INF 或者 META-INF 路径下的资源，ack 确认请求。然后调用 wrapperPipeline 的 valve 链表。

```java
    @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException {

        // Disallow any direct access to resources under WEB-INF or META-INF
        // 禁止访问 WEB-INF 或者 META-INF 路径下的资源
        MessageBytes requestPathMB = request.getRequestPathMB();
        if ((requestPathMB.startsWithIgnoreCase("/META-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/META-INF"))
                || (requestPathMB.startsWithIgnoreCase("/WEB-INF/", 0))
                || (requestPathMB.equalsIgnoreCase("/WEB-INF"))) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Select the Wrapper to be used for this Request
        Wrapper wrapper = request.getWrapper();
        if (wrapper == null || wrapper.isUnavailable()) {
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Acknowledge the request
        // ack 确认请求
        try {
            response.sendAcknowledgement(ContinueResponseTiming.IMMEDIATELY);
        } catch (IOException ioe) {
            container.getLogger().error(sm.getString(
                    "standardContextValve.acknowledgeException"), ioe);
            request.setAttribute(RequestDispatcher.ERROR_EXCEPTION, ioe);
            response.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
            return;
        }

        if (request.isAsyncSupported()) {
            request.setAsyncSupported(wrapper.getPipeline().isAsyncSupported());
        }
        // 调用 wrapperPipeline 的 valve 链表
        wrapper.getPipeline().getFirst().invoke(request, response);
    }
```

### StandardWrapperValve#invoke 方法

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



## FilterChain 接口和 Filter 接口

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

### ApplicationFilterChain#doFilter 方法

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

### ApplicationFilterChain#internalDoFilter 方法

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

