# tomcat-handler 源码分析



## 说明

1. 本文基于 tomcat 8.5.x 编写。
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## handler 模块

1. 在 tomcat 的主从 reactor 模式中，hanlder 模块指请求数据从 handler 到 servlet 的流程。



## Processor 接口

org.apache.coyote.Processor

重点注意 process, getRequest 和 recycle 方法。

```java
/**
 * Common interface for processors of all protocols.
 */
public interface Processor {
    SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status) throws IOException;

    UpgradeToken getUpgradeToken();

    boolean isUpgrade();
    
    boolean isAsync();

    void timeoutAsync(long now);

    Request getRequest();

    void recycle();

    void setSslSupport(SSLSupport sslSupport);

    ByteBuffer getLeftoverInput();

    void pause();

    boolean checkAsyncTimeoutGeneration();
}
```



## AbstractProcessorLight#process 方法

org.apache.coyote.AbstractProcessorLight#process

模板方法，在父类中维护一个 SocketState 状态机，根据状态执行对应方法。http 请求和 service 方法相关，而 websocket 和 dispatch 方法相关。

1. do while 进入 SocketState 状态机。
2. webSocket 才会用到 dispatch 的逻辑，对于 http 请求无需关心。
3. AbstractProcessorLight#service 方法的子类实现处理 http 请求。

```java
    /**
     * 模板方法，在父类中维护一个 SocketState 状态机，根据状态执行对应方法。http 请求和 service 方法相关，而 websocket 和 dispatch 方法相关。
     */
	@Override
    public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
            throws IOException {

        SocketState state = SocketState.CLOSED;
        Iterator<DispatchType> dispatches = null;
        // do while 进入 SocketState 状态机
        do {
            if (dispatches != null) {
                // ...
                // webSocket 才会用到 dispatch 的逻辑，对于 http 请求无需关心
                state = dispatch(nextDispatch.getSocketStatus());
                if (!dispatches.hasNext()) {
                    state = checkForPipelinedData(state, socketWrapper);
                }
            } else if (status == SocketEvent.DISCONNECT) {
                // Do nothing here, just wait for it to get recycled
            } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
                state = dispatch(status);
                state = checkForPipelinedData(state, socketWrapper);
            } else if (status == SocketEvent.OPEN_WRITE) {
                // Extra write event likely after async, ignore
                state = SocketState.LONG;
            } else if (status == SocketEvent.OPEN_READ) {
                // AbstractProcessorLight#service 方法的子类实现处理 http 请求
                state = service(socketWrapper);
            } else if (status == SocketEvent.CONNECT_FAIL) {
                logAccess(socketWrapper);
            } else {
                // Default to closing the socket if the SocketEvent passed in
                // is not consistent with the current state of the Processor
                state = SocketState.CLOSED;
            }
            // ...
        } while (state == SocketState.ASYNC_END ||
                dispatches != null && state != SocketState.CLOSED);

        return state;
    }
```



## Http11Processor#service 方法

org.apache.coyote.http11.Http11Processor#service

处理 http 请求。

1. 初始化一些参数，验证设置 request，SocketState 判断返回等流程。
2. 获得 adapter 实例，调用 Adapter#service 方法。这里的 adapter 对象实例是 CoyoteAdapter 类型。

```java
	@Override
    public SocketState service(SocketWrapperBase<?> socketWrapper)
        throws IOException {
        // 初始化一些参数，略过
        // ...

        while (!getErrorState().isError() && keepAlive && !isAsync() && upgradeToken == null &&
                sendfileState == SendfileState.DONE && !endpoint.isPaused()) {
            // Parsing the request header
            // 验证设置 request 等流程略过
            // ...

            // Process the request in the adapter
            if (getErrorState().isIoAllowed()) {
                try {
                    rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                    // 获得 adapter 实例，调用 Adapter#service 方法。这里的 adapter 对象实例是 CoyoteAdapter 类型。
                    getAdapter().service(request, response);
                    if(keepAlive && !getErrorState().isError() && !isAsync() &&
                            statusDropsConnection(response.getStatus())) {
                        setErrorState(ErrorState.CLOSE_CLEAN, null);
                    }
                } catch (InterruptedIOException e) {
                    setErrorState(ErrorState.CLOSE_CONNECTION_NOW, e);
                } catch (HeadersTooLargeException e) {
                    log.error(sm.getString("http11processor.request.process"), e);
                    // The response should not have been committed but check it
                    // anyway to be safe
                    if (response.isCommitted()) {
                        setErrorState(ErrorState.CLOSE_NOW, e);
                    } else {
                        response.reset();
                        response.setStatus(500);
                        setErrorState(ErrorState.CLOSE_CLEAN, e);
                        response.setHeader("Connection", "close"); // TODO: Remove
                    }
                } catch (Throwable t) {
                    ExceptionUtils.handleThrowable(t);
                    log.error(sm.getString("http11processor.request.process"), t);
                    // 500 - Internal Server Error
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, t);
                    getAdapter().log(request, response, 0);
                }
            }
            // SocketState 判断略过
            // ...
            sendfileState = processSendfile(socketWrapper);
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

        // SocketState 返回略过
        // ...
    }
```



## CoyoteAdapter#service 方法

org.apache.catalina.connector.CoyoteAdapter#service

CoyoteAdapter 适配器，适配了 connector 和 container，使 Connector 能调用到 Container。在这里面也完成了Request 和 Response 的类型转换，将 connector 模块中的 Request/Response 转换成了 container 中的。

```java
	@Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {
        // 将 org.apache.coyote 包下的 Request 和 Response, 转换成 org.apache.catalina.connector 包下的类型
        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }
        
        // ...
        
        try {
            // Parse and set Catalina and configuration specific
            // request parameters
            // 解析并设置 Catalina 核心属性和特定请求配置，如 host, context。根据请求路径来判断由哪个 StandardWrapper 处理这个请求也是在这里面完成的。
            postParseSuccess = postParseRequest(req, request, res, response);
            if (postParseSuccess) {
                //check valves if we support async
                request.setAsyncSupported(
                        connector.getService().getContainer().getPipeline().isAsyncSupported());
                // Calling the container
                // 调用进 Container 中，这里是关键，由 Connector 模块调用进了 Container，完成两者的适配
                connector.getService().getContainer().getPipeline().getFirst().invoke(
                        request, response);
            }
            // ...
        } catch (IOException e) {
            // Ignore
        } finally {
            // ...
        }
    }
```



## tomcat 的管道

1. tomcat 由 connector 和 container 两部分组成，当请求过来时 connector 先将请求包装为 request，然后传递给 container 进行处理，最终返回给请求方。
2. tomcat-container 按照包含关系一共有 4 个容器: StandardEngine, StandardHost, StandardContext, StandardWrapper. 
3. wrapper 表示一个 servlet，context 表示一个 web 应用程序，一个 web 应用程序中可能有多个 servlet。host 表示一个虚拟主机，一个虚拟主机里可能运行着多个 web 应用程序。engine 控制一个虚拟主机的生命周期。

2. container 处理的第一层就是 engine 容器，engine 容器不会直接调用 host 容器去处理请求，请求在 4 个容器中传递依靠管道。调用 pipeline 组件传递请求，跟 pipeline 相关的，还有个也是 container 内部的组件，叫做 valve 组件。

在Catalina中，我们有4种容器，每个容器都有自己的Pipeline组件，每个Pipeline组件上至少会设定一个Valve(阀门)，这个Valve我们称之为BaseValve（基础阀）。基础阀的作用是连接当前容器的下一个容器(通常是自己的自容器),可以说基础阀是两个容器之间的桥梁。

Pipeline定义对应的接口Pipeline,标准实现了StandardPipeline。Valve定义对应的接口Valve,抽象实现类ValveBase,4个容器对应基础阀门分别是StandardEngineValve,StandardHostValve,StandardContextValve,StandardWrapperValve。
