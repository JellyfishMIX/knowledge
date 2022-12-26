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



## service 方法

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

