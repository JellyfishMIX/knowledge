# guava cache 源码分析 -- builder 模式



## 说明

1. 本文基于 guava 31.1-jre 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)
4. 本文默认已经了解 builder 模式。本文所讲 Product 类，Product 类的接口，Builder 类均是 builder 模式的概念。



## Cache 接口

com.google.common.cache.Cache

guava 对 cache 基本能力的抽象。

### 接口签名

```java
/**
 * A semi-persistent mapping from keys to values. Cache entries are manually added using {@link
 * #get(Object, Callable)} or {@link #put(Object, Object)}, and are stored in the cache until either
 * evicted or manually invalidated. The common way to build instances is using {@link CacheBuilder}.
 *
 * <p>Implementations of this interface are expected to be thread-safe, and can be safely accessed
 * by multiple concurrent threads.
 *
 * guava 对 cache 基本能力的抽象
 *
 * @param <K> the type of the cache's keys, which are not permitted to be null
 * @param <V> the type of the cache's values, which are not permitted to be null
 * @author Charles Fry
 * @since 10.0
 */
@DoNotMock("Use CacheBuilder.newBuilder().build()")
@GwtCompatible
@ElementTypesAreNonnullByDefault
public interface Cache<K, V>
```

getIfPresent, get, getAllPresent 方法读取 cache 中的数据。

```java
  /**
   * Returns the value associated with {@code key} in this cache, or {@code null} if there is no
   * cached value for {@code key}.
   *
   * 如果 key 命中缓存，返回对应的 value
   * 如果 key 未命中缓存，返回 null
   *
   * @since 11.0
   */
  @CheckForNull
  @CanIgnoreReturnValue // TODO(b/27479612): consider removing this?
  V getIfPresent(@CompatibleWith("K") Object key);

  /**
   * 如果 key 命中缓存，则通过回调方法 loader 返回对应的 value，且在 loader 执行完成之前，该缓存的可见状态不会被修改。
   * 当加载缓存的时候，如果遇到一个受检异常 checked exception，会抛出 ExecutionException
   * 当加载缓存的时候，如果遇到一个非受检异常 unchecked exception，会抛出 UncheckedExecutionException
   * 当加载缓存的时候，如果遇到一个错误，会抛出 ExecutionError
   */
  @CanIgnoreReturnValue // TODO(b/27479612): consider removing this
  V get(K key, Callable<? extends V> loader) throws ExecutionException;

  /**
   * 入参是多个 key，Iterable 类型
   * 返回值是一个 ImmutableMap<K, V>，每个入参 key 及对应的 value
   */
  ImmutableMap<K, V> getAllPresent(Iterable<? extends Object> keys);
```

put, putAll 方法向 cache 中添加元素。

```java
  /**
   * 添加一个 key-value，如果 key 已存在，value 会覆盖
   */
  void put(K key, V value);

  /**
   * 批量添加 key-value，如果 key 已存在，value 会覆盖
   */
  void putAll(Map<? extends K, ? extends V> m);
```

invalidate, invalidateAll(无参)/(有参) 方法，删除 cache 中的缓存。

```java
  /**
   * 删除 key 对应的缓存
   */
  void invalidate(@CompatibleWith("K") Object key);

  /**
   * 批量删除 key 对应的缓存
   */
  // For discussion of <? extends Object>, see getAllPresent.
  void invalidateAll(Iterable<? extends Object> keys);

  /**
   * 删除 cache 中所有的缓存项
   */
  void invalidateAll();
```

size 方法，返回 cache 中缓存项的数量。

```java
  /**
   * 返回 cache 中缓存项的数量
   */
  long size();
```

stats 方法，返回此 cache 的累积统计信息的当前快照。所有统计信息初始化时均为 0，在 cache 的整个生命周期内单调增加。

```java
  /**
   * Returns a current snapshot of this cache's cumulative statistics, or a set of default values if
   * the cache is not recording statistics. All statistics begin at zero and never decrease over the
   * lifetime of the cache.
   *
   * <p><b>Warning:</b> this cache may not be recording statistical data. For example, a cache
   * created using {@link CacheBuilder} only does so if the {@link CacheBuilder#recordStats} method
   * was called. If statistics are not being recorded, a {@code CacheStats} instance with zero for
   * all values is returned.
   *
   * 返回此 cache 的累积统计信息的当前快照。
   * 所有统计信息初始化时均为 0，在 cache 的整个生命周期内单调增加。
   *
   */
  CacheStats stats();
```

asMap 方法，返回一个线程安全的 map，此 map 是 cache 的视图。对 map 的修改将直接影响 cache。

```java
  /**
   * Returns a view of the entries stored in this cache as a thread-safe map. Modifications made to
   * the map directly affect the cache.
   *
   * <p>Iterators from the returned map are at least <i>weakly consistent</i>: they are safe for
   * concurrent use, but if the cache is modified (including by eviction) after the iterator is
   * created, it is undefined which of the changes (if any) will be reflected in that iterator.
   *
   * 返回一个线程安全的 map，此 map 是 cache 的视图。对 map 的修改将直接影响 cache
   *
   */
  ConcurrentMap<K, V> asMap();
```

cleanUp 方法，执行 cache 所需要的任何待执行维护操作。究竟执行哪些动作与实现相关。

```java
  /**
   * Performs any pending maintenance operations needed by the cache. Exactly which activities are
   * performed -- if any -- is implementation-dependent.
   *
   * 执行 cache 所需要的任何待执行维护操作。究竟执行哪些动作与实现相关。
   */
  void cleanUp();
}
```



## guava cache 的使用

api 的使用是典型的 builder 模式。

```java
  public LoadingCache<String, String> serviceCache =
      CacheBuilder.newBuilder().maximumSize(20000).expireAfterWrite(10, TimeUnit.SECONDS)
          .recordStats().build(new CacheLoader<String, String>() {
        @Override
        public String load(String key) {
          return service.query(key);
        }
      });
```

1. 首先是 CacheBuilder.newBuilder() 创建一个 CacheBuilder。
2. 然后对所需属性进行赋值。
3. 最后调用 CacheBuilder 的 build() 构建方法，构建 LoadingCache 的实现类。



## guava cache 的 builder 模式

1. guava cache 的 builder 模式，Product 类有两个，一个是 LocalCache，一个是 LocalLoadingCache。LocalCache 用于实现内部机制，LocalLoadingCache 用于对外暴露。

2. 对于 LocalLoadingCache 这样对外暴露的 Product 类，LoadingCache 接口抽象(规范)了他们的能力。



## 内部使用的 Product 类--LocalCache

com.google.common.cache.LocalCache

builder 模式，内部使用的 Product 类。

```java
/**
 * The concurrent hash map implementation built by {@link CacheBuilder}.
 *
 * <p>This implementation is heavily derived from revision 1.96 of <a
 * href="http://tinyurl.com/ConcurrentHashMap">ConcurrentHashMap.java</a>.
 *
 * builder 模式，内部使用的 Product 类
 *
 * @author Charles Fry
 * @author Bob Lee ({@code com.google.common.collect.MapMaker})
 * @author Doug Lea ({@code ConcurrentHashMap})
 */
@SuppressWarnings({
        "GoodTime", // lots of violations (nanosecond math)
        "nullness", // too much trouble for the payoff
})
@GwtCompatible(emulated = true)
// TODO(cpovirk): Annotate for nullness.
class LocalCache<K, V> extends AbstractMap<K, V> implements ConcurrentMap<K, V>
```

### LocalCache 构造方法

com.google.common.cache.LocalCache#LocalCache

1. LocalCache 的构造方法，传入一个 builder。
2. LocalCache 是 Product，CacheBuilder 是 builder。

```java
    /**
     * Creates a new, empty map with the specified strategy, initial capacity and concurrency level.
     *
     * LocalCache 的构造方法，传入一个 builder
     * LocalCache 是 Product，CacheBuilder 是 builder。
     */
    LocalCache(
            CacheBuilder<? super K, ? super V> builder, @CheckForNull CacheLoader<? super K, V> loader) {
        concurrencyLevel = Math.min(builder.getConcurrencyLevel(), MAX_SEGMENTS);

        keyStrength = builder.getKeyStrength();
        valueStrength = builder.getValueStrength();

        keyEquivalence = builder.getKeyEquivalence();
        valueEquivalence = builder.getValueEquivalence();

        maxWeight = builder.getMaximumWeight();
        weigher = builder.getWeigher();
        expireAfterAccessNanos = builder.getExpireAfterAccessNanos();
        expireAfterWriteNanos = builder.getExpireAfterWriteNanos();
        refreshNanos = builder.getRefreshNanos();

        removalListener = builder.getRemovalListener();
        removalNotificationQueue =
                (removalListener == NullListener.INSTANCE)
                        ? LocalCache.discardingQueue()
                        : new ConcurrentLinkedQueue<>();

        ticker = builder.getTicker(recordsTime());
        entryFactory = EntryFactory.getFactory(keyStrength, usesAccessEntries(), usesWriteEntries());
        globalStatsCounter = builder.getStatsCounterSupplier().get();
        defaultLoader = loader;

		// ......(更多内容省略)
    }
```



## 对外暴露 Product 类的基础实现类--LocalManualCache

com.google.common.cache.LocalCache.LocalManualCache

1. LocalManualCache 实现了 Cache，具有 Cache 的能力，作为 Cache 的基础实现类。
2. LocalManualCache 的子类可以继承后重写，实现定制化逻辑。如果不重写，也可以直接使用继承过来的能力，提高代码复用度。

```java
    /**
     * LocalManualCache 实现了 Cache，具有 Cache 的能力，作为 Cache 的基础实现类。
     * LocalManualCache 的子类可以继承后重写，实现定制化逻辑。如果不重写，也可以直接使用继承过来的能力，提高代码复用度。
     *
     * @param <K>
     * @param <V>
     */
    static class LocalManualCache<K, V> implements Cache<K, V>, Serializable
```



## 对外暴露的 Product 类--LocalLoadingCache

com.google.common.cache.LocalCache.LocalLoadingCache

1. builder 模式，对外暴露的 Product 类。
2. LocalLoadingCache 继承了 LocalManualCache，因此获得了 Cache 的能力。LocalLoadingCache 可以选择地重写 LocalManualCache 中的方法，实现定制化逻辑。如果不重写，也可以直接使用继承过来的能力，提高代码复用度。

```java
    /**
     * builder 模式，对外暴露的 Product 类。
     * LocalLoadingCache 继承了 LocalManualCache，因此获得了 Cache 的能力。
     * LocalLoadingCache 可以选择地重写 LocalManualCache 中的方法，实现定制化逻辑。如果不重写，也可以直接使用继承过来的能力，提高代码复用度。
     *
     * @param <K>
     * @param <V>
     */
	static class LocalLoadingCache<K, V> extends LocalManualCache<K, V>
            implements LoadingCache<K, V> {

        LocalLoadingCache(
                CacheBuilder<? super K, ? super V> builder, CacheLoader<? super K, V> loader) {
            super(new LocalCache<>(builder, checkNotNull(loader)));
        }
        // ......(更多内容省略)
    }
```



## Product 类的接口--LoadingCache

com.google.common.cache.LoadingCache

1. LocalCache 中所有对外暴露的 Product 类，都实现了这个接口。
2. 例如 LocalLoadingCache 就是实现了这个接口。

```java
/**
 * A semi-persistent mapping from keys to values. Values are automatically loaded by the cache, and
 * are stored in the cache until either evicted or manually invalidated. The common way to build
 * instances is using {@link CacheBuilder}.
 *
 * <p>Implementations of this interface are expected to be thread-safe, and can be safely accessed
 * by multiple concurrent threads.
 *
 * <p>When evaluated as a {@link Function}, a cache yields the same result as invoking {@link
 * #getUnchecked}.
 *
 * LocalCache 中所有对外暴露的 Product 类，都实现了这个接口。
 * 例如 LocalLoadingCache 就是实现了这个接口。
 *
 * @param <K> the type of the cache's keys, which are not permitted to be null
 * @param <V> the type of the cache's values, which are not permitted to be null
 * @author Charles Fry
 * @since 11.0
 */
@GwtCompatible
@ElementTypesAreNonnullByDefault
public interface LoadingCache<K, V> extends Cache<K, V>, Function<K, V> {

  @CanIgnoreReturnValue
  V get(K key) throws ExecutionException;

  @CanIgnoreReturnValue
  V getUnchecked(K key);

  @CanIgnoreReturnValue
  ImmutableMap<K, V> getAll(Iterable<? extends K> keys) throws ExecutionException;

  @Deprecated
  @Override
  V apply(K key);

  void refresh(K key);

  @Override
  ConcurrentMap<K, V> asMap();
}
```



## Builder 类--CacheBuilder

com.google.common.cache.CacheBuilder

### 构建方法 build

com.google.common.cache.CacheBuilder#build(com.google.common.cache.CacheLoader<? super K1,V1>)

1. builder 的构建方法，通过此方法可以构建 Product: LocalCache, LocalLoadingCache。

2. 调用 LocalCache.LocalLoadingCache 的构造方法，构造方法中会创建一个 LocalCache 对象。然后使用 LocalCache 对象，创建一个 LocalLoadingCache。

3. 可以认为 build 出来的是对外暴露的 Product 类--LocalLoadingCache，LocalLoadingCache 使用了内部的 Product 类 LocalCache。

```java
    /**
     * Builds a cache, which either returns an already-loaded value for a given key or atomically
     * computes or retrieves it using the supplied {@code CacheLoader}. If another thread is currently
     * loading the value for this key, simply waits for that thread to finish and returns its loaded
     * value. Note that multiple threads can concurrently load values for distinct keys.
     *
     * <p>This method does not alter the state of this {@code CacheBuilder} instance, so it can be
     * invoked again to create multiple independent caches.
     *
     * builder 的构建方法，通过此方法可以构建 Product: LocalCache, LocalLoadingCache。
     * 调用 LocalCache.LocalLoadingCache 的构造方法，构造方法中会创建一个 LocalCache 对象。然后使用 LocalCache 对象，创建一个 LocalLoadingCache。
     * 可以认为 build 出来的是对外暴露的 Product 类--LocalLoadingCache，LocalLoadingCache 使用了内部的 Product 类 LocalCache。
     *
     * @param loader the cache loader used to obtain new values
     * @return a cache having the requested features
     */
    public <K1 extends K, V1 extends V> LoadingCache<K1, V1> build(
            CacheLoader<? super K1, V1> loader) {
        checkWeightWithWeigher();
        return new LocalCache.LocalLoadingCache<>(this, loader);
    }
```

