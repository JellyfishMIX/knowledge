# CompletableFuture 源码分析



## 说明

1. 本文基于 jdk 8 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## asyncPool(自带的线程池)

请见注释：

```java
	/**
     * Default executor -- ForkJoinPool.commonPool() unless it cannot
     * support parallelism.
     *
     * 自带的线程池，来自于 ForkJoinPool 的 commonPool。
     * ForkJoinPool 的 commonPool 在 static {} 里初始化，用了 Unsafe 这个闭源类，会根据机器的硬件配置去初始化 commonPool
     * 业务一般不用自带的线程池，会用业务自己传入的线程池。ForkJoinPool 的 commonPool 只是作为框架提供的默认实现，不推荐使用。
     */
    private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```



## thenApply(同步), thenApplyAsync(异步)

thenApply 方法同步地执行一个函数，thenApplyAsync 异步地执行一个函数（异步即不会占用当前线程，会使用其他线程）。

```java
	public <U> CompletableFuture<U> thenApply(
        Function<? super T,? extends U> fn) {
        return uniApplyStage(null, fn);
    }

    public <U> CompletableFuture<U> thenApplyAsync(
        Function<? super T,? extends U> fn) {
        return uniApplyStage(asyncPool, fn);
    }

    public <U> CompletableFuture<U> thenApplyAsync(
        Function<? super T,? extends U> fn, Executor executor) {
        return uniApplyStage(screenExecutor(executor), fn);
    }
```

可以看到，不管是 thenApply 还是 thenApplyAsync 都调用了 uniApplyStage 方法，uniApplyStage 方法如下：

```java
	private <V> CompletableFuture<V> uniApplyStage(
        Executor e, Function<? super T,? extends V> f) {
        // 针对传入的函数判空校验
        if (f == null) throw new NullPointerException();
        // 将要返回的 CompletableFuture 实例
        CompletableFuture<V> d =  new CompletableFuture<V>();
        // Async 直接进入 if 分支，sync 则执行返回值的 uniApply 尝试获取结果（因为 Async 会传入线程池，通过判断入参有无线程池，来判断是否异步）
        if (e != null || !d.uniApply(this, f, null)) {
            UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
            push(c);
            c.tryFire(SYNC);
        }
        return d;
    }
```

uniApplyStage 方法及其调用的方法，由于我的水平不足，暂未有能力解析，日后将完善。记录下质量较高的学习文章，分析时可供参考：[深入解读CompletableFuture源码与原理 - CoderBruis - CSDN](https://blog.csdn.net/CoderBruis/article/details/103181520)



1. 如果调用方不依赖响应结果，dubbo 接口可以使用异步处理的方式，加快响应速度，避免让调用方长时间等待。例如以下写法：

   ```java
   	public void callback(String requestId, SearchResponse searchResponse) {
           CompletableFuture.runAsync(() -> {...}, QFARE_CALL_BACK_EXECUTOR);
       }
   ```

   1. 这种写法常见于 dubbo 回调方法中，因为触发回调方法的那方，不依赖于回调方法的执行结果，只是去做触发的动作。
   2. 此外，如果调用 provider 的 rpc 方法，consumer 可以选择不等待本次响应结果的返回，而是把一个回调方法当作入参传递给 provider，此时 provider 也可以使用异步处理的方式，来加快响应速度，避免 consumer 长时间等待。

2. CompletableFuture 的两个 api 可以执行异步任务，supplyAsync (有返回值)，runAsync (无返回值)。注意需要传入一个自定义的线程池。

3. CompletableFuture 的 get() 和 getNow(T valueIfAbsent) 方法的区别：get() 会阻塞住，等待 CompletableFuture 有了 result 后才会返回。getNow(T valueIfAbsent) 方法不会阻塞，如果 CompletableFuture 还没有 result，则会把入参 valueIfAbsent 返回。



并行计算的组合写法。

```java
		// 代理商政策做切分，切分成多批。每批的大小由 qconfig 控制。
        List<List<String>> clientPartitions = Lists.partition(clients, uniqConfig.getClientMatchSize());
        // 分批计算 policy 政策与 fareInfo 的匹配
        List<CompletableFuture<List<String>>> noSearchClientFuture = clientPartitions.stream()
                // 多批用多线程去做匹配计算，并行效率更高
                .map(clientTaskList -> CompletableFuture.supplyAsync(() ->
                        clientMatch(searchContext, fareInfoWrapper, status, clientTaskList, policyResult, debugProcess), CLIENT_MATCH_TASK_EXECUTOR)
                )
                .collect(Collectors.toList());

        // 收束点，多线程转单线程。等多线程匹配计算完毕后，没有算出报价的代理商，聚合为一个 list 返回。
        return CompletableFuture.allOf(noSearchClientFuture.toArray(new CompletableFuture[0]))
                .thenApply(v -> noSearchClientFuture.stream()
                        .flatMap(future -> future.getNow(Collections.emptyList()).stream())
                        .collect(Collectors.toList())
                );
```

大集合切分成多个小集合，使用 CompletableFuture.supplyAsync 多线程并行计算提高效率，最后用 CompletableFuture.allOf 收束，多线程转单线程(异步转同步)处理计算结果。



## thenApply 和 thenCompose 的区别

1. thenApply 接收一个函数作为参数，使用此函数处理上一个 CompletableFuture 计算步骤的结果，此函数的返回结果作为本次计算步骤的 result。
2. thenCompose 接收一个函数作为参数，此函数需返回 CompletableFuture 实例。此函数的入参是先前 CompletableFuture 计算步骤的结果。
3. thenApply 的入参函数转换的是 result 泛型的类型，返回的是计算结果 result。最终 thenApply 返回的还是先前的 CompletableFuture。
4. thenCompose 的入参函数使用上一个计算步骤的 result，最终会生成一个新的 CompletableFuture 并返回。

对 CompletableFuture 的 api 讲解比较清楚的一篇文章 CompletableFuture中whenComplete()和thenApply() 区别? - 程序员子龙的回答 - 知乎
https://www.zhihu.com/question/433003386/answer/2295290317



## 哪个线程执行 CompletableFuture 的任务和回调

### CompletableFuture 提交的任务

1. 异步执行的 supplyAsync(() -> {}) 方法，类似于 ExecutorService.submit()。提交任务至线程池后返回 CompletableFuture。
2. 未显式指定线程池时，使用默认的 ForkJoinPool.commonPool()，一个 ForkJoinPool 的类属性，单例。即 ForkJoinPool.commonPool() 是一个全局单例的线程池，线程池配置为硬编码与当前本地机器有关，配置不可手动修改，只是一个默认的实现并不推荐使用。
3. 这导致同一 JVM 上的所有应用各代码位置共享同一个线程池，此线程池配置不可修改不可控，没有资源隔离。因此应该显式指定自定义的线程池，替换掉默认的线程池。

```java
// 未显式指定线程池，使用默认线程池 ForkJoinPool.commonPool()
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try (
                InputStream is = new URL("http://www.baidu.com").openStream()) {
                return IOUtils.toString(is, StandardCharsets.UTF_8);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        });

// 显式指定线程池，使用自定义的线程池
private static ExecutorService diyThreadPoolExecutor = new ThreadPoolExecutor(...);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try (
                InputStream is = new URL("http://www.baidu.com").openStream()) {
                return IOUtils.toString(is, StandardCharsets.UTF_8);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }, diyThreadPoolExecutor);
```

### 处理 CompletableFuture 结果的回调函数

1. 当 future 有了结果后，thenApply 中的回调函数需要被执行，此回调函数由何线程执行看具体情况。

2. 如果执行原始任务的线程池未关闭，则使用原始任务所用的线程池，执行回调函数。如果原始任务所用的线程池已关闭，则在调用 thenApply 方法的 main 线程中同步地执行回调函数。

```java
private static ExecutorService diyThreadPoolExecutor = new ThreadPoolExecutor(...);
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try (
                InputStream is = new URL("http://www.baidu.com").openStream()) {
                return IOUtils.toString(is, StandardCharsets.UTF_8);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }, diyThreadPoolExecutor);

CompletableFuture<Integer> intFuture = future.thenApply(s -> s.length());
```



## CompletableFuture 应用场景

1. 通常用在需要返回值的异步计算，如果不需要返回值，直接使用线程池即可。
2. 需要返回值的异步计算返回 Future，CompletableFuture 作为实现者提供了友好的 api。