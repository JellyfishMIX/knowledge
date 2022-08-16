# CompletableFuture 源码分析



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