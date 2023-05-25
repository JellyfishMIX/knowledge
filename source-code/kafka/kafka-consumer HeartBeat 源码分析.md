# kafka-consumer HeartBeat 源码分析



## 说明

1. 本文基于 jdk 8, kafka 0.10.0 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## HeartBeat 作用

1. 消费者定期向服务端的 GroupCoordinator 发送 HeartbeatRequest 来确定彼此在线。
2. 服务端可以通过 HeartbeatResponse 给 consumer 返回 IllegalGeneration异常，通知 consumer 进行 Rebalance。



## HeartbeatTask -- 定时任务

org.apache.kafka.clients.consumer.internals.AbstractCoordinator.HeartbeatTask#run

1. 检查三个发送 HeartbeatRequest 的必要条件，如果不符合，就不会再执行 HeartbeatTask，等待后续调用 reset() 方法重启 HeartbeatTask 任务。
   1. GroupCoordinator 已经确定且已连接。
   2. 不处于正在等待 Partition 分配结果的状态。
   3. 之前的 HeartbeatRequest 请求正常收到响应且没有过期。
2. 判断 HeartbeatResponse 是否超时。
3. 还没到发送心跳请求的时间，重新调度任务的执行。
4. 到了发送心跳请求的时间，更新发送 HeartbeatRequest 的时间。
5. 发送异步请求 HeartbeatRequest，为请求添加 listener 处理异步请求结果，申请下次心跳调度任务。

```java
    private class HeartbeatTask implements DelayedTask {

		@Override
        public void run(final long now) {
            /*
             * 检查三个发送 HeartbeatRequest 的必要条件，如果不符合，就不会再执行 HeartbeatTask，等待后续调用 reset() 方法重启 HeartbeatTask 任务
             * 1. GroupCoordinator 已经确定且已连接
             * 2. 不处于正在等待 Partition 分配结果的状态
             * 3. 之前的 HeartbeatRequest 请求正常收到响应且没有过期
             */
            if (generation < 0 || needRejoin() || coordinatorUnknown()) {
                // no need to send the heartbeat we're not using auto-assignment or if we are
                // awaiting a rebalance
                return;
            }

            // 判断 HeartbeatResponse 是否超时
            if (heartbeat.sessionTimeoutExpired(now)) {
                // we haven't received a successful heartbeat in one session interval
                // so mark the coordinator dead
                coordinatorDead();
                return;
            }

            // 还没到发送心跳请求的时间，重新调度任务的执行
            if (!heartbeat.shouldHeartbeat(now)) {
                // we don't need to heartbeat now, so reschedule for when we do
                client.schedule(this, now + heartbeat.timeToNextHeartbeat(now));
            } else {
                // 更新发送 HeartbeatRequest 的时间
                heartbeat.sentHeartbeat(now);
                requestInFlight = true;

                // 发送异步请求 HeartbeatRequest
                RequestFuture<Void> future = sendHeartbeatRequest();
                // 为请求添加 listener 处理异步请求结果
                future.addListener(new RequestFutureListener<Void>() {
                    @Override
                    public void onSuccess(Void value) {
                        requestInFlight = false;
                        long now = time.milliseconds();
                        heartbeat.receiveHeartbeat(now);
                        long nextHeartbeatTime = now + heartbeat.timeToNextHeartbeat(now);
                        client.schedule(HeartbeatTask.this, nextHeartbeatTime);
                    }

                    @Override
                    public void onFailure(RuntimeException e) {
                        requestInFlight = false;
                        client.schedule(HeartbeatTask.this, time.milliseconds() + retryBackoffMs);
                    }
                });
            }
        }
    }
```



## HeartbeatResponse 处理

org.apache.kafka.clients.consumer.internals.AbstractCoordinator#sendHeartbeatRequest

1. 发送一个异步请求，返回值是 RequestFuture，同时调用 RequestFuture#compose 方法给 RequestFuture 绑定一个 HeartbeatCompletionHandler，负责处理请求结果。

```java
	public RequestFuture<Void> sendHeartbeatRequest() {
        HeartbeatRequest req = new HeartbeatRequest(this.groupId, this.generation, this.memberId);
        return client.send(coordinator, ApiKeys.HEARTBEAT, req)
                .compose(new HeartbeatCompletionHandler());
    }
```

### RequestFuture#compose -- 给 RequestFuture 绑定 listener

org.apache.kafka.clients.consumer.internals.RequestFuture#compose

1. 给 RequestFuture 绑定一个 listener(需要是 RequestFutureAdapter 的实现类)，负责处理请求结果。

```java
	public <S> RequestFuture<S> compose(final RequestFutureAdapter<T, S> adapter) {
        final RequestFuture<S> adapted = new RequestFuture<S>();
        addListener(new RequestFutureListener<T>() {
            @Override
            public void onSuccess(T value) {
                adapter.onSuccess(value, adapted);
            }

            @Override
            public void onFailure(RuntimeException e) {
                adapter.onFailure(e, adapted);
            }
        });
        return adapted;
    }
```

### HeartbeatCompletionHandler -- 负责处理 HeartbeatResponse 的回调函数

HeartbeatCompletionHandler 继承自 CoordinatorResponseHandler。

![img](https://image-hosting.jellyfishmix.com/20230410214357)

### CoordinatorResponseHandler -- 负责处理 CoordinatorResponse 的抽象类

1. CoordinatorResponseHandler 是一个抽象类，有 parse() 和 handle() 两个抽象方法，parse() 方法对 ClientResponse 进行解析，得到指定类型的响应。handle() 对解析后的响应进行处理。
2. CoordinatorResponseHandler 实现了 RequestFuture 抽象类的 onSuccess() 方法和 onFailure 方法。
3. CoordinatorResponseHandler 应用了模板方法模式，由父类方法定义操作流程，子类根据需求个性化实现流程中的抽象方法。

```java
    protected abstract class CoordinatorResponseHandler<R, T>
            extends RequestFutureAdapter<ClientResponse, T> {
        protected ClientResponse response;

        public abstract R parse(ClientResponse response);

        public abstract void handle(R response, RequestFuture<T> future);

        @Override
        public void onFailure(RuntimeException e, RequestFuture<T> future) {
            // mark the coordinator as dead
            if (e instanceof DisconnectException)
                coordinatorDead();
            future.raise(e);
        }

        @Override
        public void onSuccess(ClientResponse clientResponse, RequestFuture<T> future) {
            try {
                this.response = clientResponse;
                R responseObj = parse(clientResponse);
                handle(responseObj, future);
            } catch (RuntimeException e) {
                if (!future.isDone())
                    future.raise(e);
            }
        }
    }
```

### HeartbeatCompletionHandler#handle 方法 -- 处理 HeartbeatResponse

org.apache.kafka.clients.consumer.internals.AbstractCoordinator.HeartbeatCompletionHandler#handle

根据 response 有如下处理:

1. 心跳正常。
2. 找不到服务端 GroupCoordinator，清空 unsent 集合中对应的请求，并重新查找对应的 GroupCoordinator。
3. 正在 rebalance, 会重新发送 JoinGroupRequest 消息。
4. 服务端 GroupCoordinator 告知 consumer 需要执行 rebalance, 发送 JoinGroupRequest。
5. 未知的 MemberId, 需要执行 rebalance, 发送 JoinGroupRequest。
6. consumerGroup 授权失败，抛出异常。
7. 其他 response 不支持，抛出异常。

```java
	private class HeartbeatCompletionHandler extends CoordinatorResponseHandler<HeartbeatResponse, Void> {
        @Override
        public HeartbeatResponse parse(ClientResponse response) {
            return new HeartbeatResponse(response.responseBody());
        }

        @Override
        public void handle(HeartbeatResponse heartbeatResponse, RequestFuture<Void> future) {
            sensors.heartbeatLatency.record(response.requestLatencyMs());
            Errors error = Errors.forCode(heartbeatResponse.errorCode());
            // 心跳正常
            if (error == Errors.NONE) {
                log.debug("Received successful heartbeat response for group {}", groupId);
                future.complete(null);
                // 找不到服务端 GroupCoordinator
            } else if (error == Errors.GROUP_COORDINATOR_NOT_AVAILABLE
                    || error == Errors.NOT_COORDINATOR_FOR_GROUP) {
                log.debug("Attempt to heart beat failed for group {} since coordinator {} is either not started or not valid.",
                        groupId, coordinator);
                // 清空 unsent 集合中对应的请求，并重新查找对应的 GroupCoordinator
                coordinatorDead();
                future.raise(error);
                // 正在 rebalance, 会重新发送 JoinGroupRequest 消息
            } else if (error == Errors.REBALANCE_IN_PROGRESS) {
                log.debug("Attempt to heart beat failed for group {} since it is rebalancing.", groupId);
                AbstractCoordinator.this.rejoinNeeded = true;
                future.raise(Errors.REBALANCE_IN_PROGRESS);
                // 服务端 GroupCoordinator 告知 consumer 需要执行 rebalance, 发送 JoinGroupRequest
            } else if (error == Errors.ILLEGAL_GENERATION) {
                log.debug("Attempt to heart beat failed for group {} since generation id is not legal.", groupId);
                AbstractCoordinator.this.rejoinNeeded = true;
                future.raise(Errors.ILLEGAL_GENERATION);
                // 未知的 MemberId, 需要执行 rebalance, 发送 JoinGroupRequest
            } else if (error == Errors.UNKNOWN_MEMBER_ID) {
                log.debug("Attempt to heart beat failed for group {} since member id is not valid.", groupId);
                memberId = JoinGroupRequest.UNKNOWN_MEMBER_ID;
                AbstractCoordinator.this.rejoinNeeded = true;
                future.raise(Errors.UNKNOWN_MEMBER_ID);
                // consumerGroup 授权失败，抛出异常
            } else if (error == Errors.GROUP_AUTHORIZATION_FAILED) {
                future.raise(new GroupAuthorizationException(groupId));
                // 其他 response 不支持，抛出异常
            } else {
                future.raise(new KafkaException("Unexpected error in heartbeat response: " + error.message()));
            }
        }
    }
```

