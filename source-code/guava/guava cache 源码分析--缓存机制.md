# guava cache 源码分析 -- 缓存机制



## 说明

1. 本文基于 guava 31.1-jre 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 缓存机制相关的数据结构

1. guava cache 内部缓存机制，使用 builder 模式中对应 Product 概念的 LocalCache 实现。

2. LocalCache 类似 jdk7 及以前的 ConcurrentHashMap，采用了分段策略，通过减小锁的粒度来提高并发。LocalCache 中数据存储在 Segment[] 中，每个 segment 包含 1 个 table 和 5 个队列。

### Segment

com.google.common.cache.LocalCache.Segment

```java
    /**
     * Segments are specialized versions of hash tables. This subclass inherits from ReentrantLock
     * opportunistically, just to simplify some locking and avoid separate construction.
     *
     * LocalCache 类似 jdk7 及以前的 ConcurrentHashMap，采用了分段策略，通过减小锁的粒度来提高并发。
     * LocalCache 中数据存储在 Segment[] 中，每个 segment 包含 1 个 table 和 5 个队列。
     */
    @SuppressWarnings("serial") // This class is never serialized.
    static class Segment<K, V> extends ReentrantLock
```

1. LocalCache 类似 jdk7 及以前的 ConcurrentHashMap，采用了分段策略，通过减小锁的粒度来提高并发。
2. LocalCache 中数据存储在 Segment[] 中，每个 segment 包含 1 个 table 和 5 个队列。

### table

1. table 是一种类数组的线程安全结构 AtomicReferenceArray，每个元素是一个 ReferenceEntry<K, V> 链表。
2. AtomicReferenceArray 是 juc 包中 Doug Lea 设计的类：一组对象引用，元素支持原子性更新。

```java
        /**
         * The per-segment table.
         *
         * 每个 segment 的 table
         */
        @CheckForNull volatile AtomicReferenceArray<ReferenceEntry<K, V>> table;
```

### ReferenceEntry

ReferenceEntry 被抽象出了接口，声明了 ReferenceEntry 拥有的能力。具体的实现类有 StrongEntry, WeakEntry, NullEntry 等，默认使用 StrongEntry 强引用。

```java
/**
 * An entry in a reference map.
 *
 * <p>Entries in the map can be in the following states:
 *
 * <p>Valid:
 *
 * <ul>
 *   <li>Live: valid key/value are set
 *   <li>Loading: loading is pending
 * </ul>
 *
 * <p>Invalid:
 *
 * <ul>
 *   <li>Expired: time expired (key/value may still be set)
 *   <li>Collected: key/value was partially collected, but not yet cleaned up
 *   <li>Unset: marked as unset, awaiting cleanup or reuse
 * </ul>
 */
@GwtIncompatible
@ElementTypesAreNonnullByDefault
interface ReferenceEntry<K, V> {
```

![image-20221017132452569](https://image-hosting.jellyfishmix.com/20221017132452.png)

### 5 个队列

这 5 个队列，前 2 个是 key, value 引用队列。后 3 个是频次队列(LRU 算法)，写记录队列，访问记录队列。

```java
    /**
     * Segments are specialized versions of hash tables. This subclass inherits from ReentrantLock
     * opportunistically, just to simplify some locking and avoid separate construction.
     *
     * LocalCache 类似 jdk7 及以前的 ConcurrentHashMap，采用了分段策略，通过减小锁的粒度来提高并发。
     * LocalCache 中数据存储在 Segment[] 中，每个 segment 包含 1 个 table 和 5 个队列。
     */
    @SuppressWarnings("serial") // This class is never serialized.
    static class Segment<K, V> extends ReentrantLock {
        // ......
        
		/**
         * The key reference queue contains entries whose keys have been garbage collected, and which
         * need to be cleaned up internally.
         *
         * key 引用队列
         */
        @CheckForNull final ReferenceQueue<K> keyReferenceQueue;

        /**
         * The value reference queue contains value references whose values have been garbage collected,
         * and which need to be cleaned up internally.
         *
         * value 引用队列
         */
        @CheckForNull final ReferenceQueue<V> valueReferenceQueue;

        /**
         * The recency queue is used to record which entries were accessed for updating the access
         * list's ordering. It is drained as a batch operation when either the DRAIN_THRESHOLD is
         * crossed or a write occurs on the segment.
         *
         * 频次队列，LRU 算法
         */
        final Queue<ReferenceEntry<K, V>> recencyQueue;

        /**
         * A counter of the number of reads since the last write, used to drain queues on a small
         * fraction of read operations.
         */
        final AtomicInteger readCount = new AtomicInteger();

        /**
         * A queue of elements currently in the map, ordered by write time. Elements are added to the
         * tail of the queue on write.
         *
         * 写记录队列
         */
        @GuardedBy("this")
        final Queue<ReferenceEntry<K, V>> writeQueue;

        /**
         * A queue of elements currently in the map, ordered by access time. Elements are added to the
         * tail of the queue on access (note that writes count as accesses).
         *
         * 访问记录队列
         */
        @GuardedBy("this")
        final Queue<ReferenceEntry<K, V>> accessQueue;
        
        // ......
    }
```

示意图：

图 1

![img](https://image-hosting.jellyfishmix.com/20221017010642.png)

图 2

![img](https://image-hosting.jellyfishmix.com/20221017010649.png)



## LocalLoadingCache#get 方法

com.google.common.cache.LocalCache.LocalLoadingCache#get

对外暴露的 Product 类 LocalLoadingCache 提供的方法，使用了内部 Product 类 localCache

```java
        /**
         * 对外暴露的 Product 类 LocalLoadingCache 提供的方法，使用了内部 Product 类 localCache
         * 
         * @param key
         * @return
         * @throws ExecutionException
         */
        @Override
        public V get(K key) throws ExecutionException {
            return localCache.getOrLoad(key);
        }
```



## LocalCache#getOrLoad 方法

com.google.common.cache.LocalCache#getOrLoad

内部 Product 类 LocalCache 的方法，默认使用 defaultLoader

```java
    /**
     * 内部 Product 类 LocalCache 的方法，默认使用 defaultLoader
     *
     * @param key
     * @return
     * @throws ExecutionException
     */
    V getOrLoad(K key) throws ExecutionException {
        return get(key, defaultLoader);
    }
```



## LocalCache#get 方法

com.google.common.cache.LocalCache#get(K, com.google.common.cache.CacheLoader<? super K,V>)

内部 Product 类 LocalCache 的方法。根据 key 的 hash 先拿 segment，再去 segment 中拿 value。

```java
    /**
     * 内部 Product 类 LocalCache 的方法
     * 根据 key 的 hash 先拿 segment，再去 segment 中拿 value
     *
     * @param key
     * @param loader
     * @return
     * @throws ExecutionException
     */
    @CanIgnoreReturnValue // TODO(b/27479612): consider removing this
    V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
        // 计算 key 的 hash
        int hash = hash(checkNotNull(key));
        // 根据 hash 计算 key 对应的 segment
        return segmentFor(hash).get(key, hash, loader);
    }
```



## LocalCache#segmentFor 方法

com.google.common.cache.LocalCache#segmentFor

这是根据 hash 计算 key 对应的 segment 的算法，不用关注。

```java
    /**
     * Returns the segment that should be used for a key with the given hash.
     * 
     * 这是根据 hash 计算 key 对应的 segment 的算法，不用关注
     *
     * @param hash the hash code for the key
     * @return the segment
     */
    Segment<K, V> segmentFor(int hash) {
        // TODO(fry): Lazily create segments?
        return segments[(hash >>> segmentShift) & segmentMask];
    }
```



## Segment#get 方法

com.google.common.cache.LocalCache.Segment#get(K, int, com.google.common.cache.CacheLoader<? super K,V>)

从 segment 中根据 key 拿 value。

1. key 和 loader 不能为空，否则抛 NullPointerException。
2. 调用 Segment#getEntry 方法，根据 key 和 hash，获取对应的 entry。
3. 调用 Segment#getLiveValue 方法获取有效的 value。
4. 如果获取到了 value，记录访问，维护 LRU 队列。statsCounter 统计信息。
5. 根据用户是否设置刷新时间(距离上次写入间隔一段时间)，进行刷新或返回旧值。
6. 如果 value 正在加载中，调用 Segment#waitForLoadingValue 方法，等待其他线程 load 完成后获取 value。
7. value 不存在或者过期，调用 Segment#lockedGetOrLoad 方法通过 loader 阻塞地加载 value。

```java
        /**
         * 从 segment 中根据 key 拿 value
         *
         * @param key
         * @param hash
         * @param loader
         * @return
         * @throws ExecutionException
         */
        @CanIgnoreReturnValue
        V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
            // key 和 loader 不能为空，否则抛 NullPointerException
            checkNotNull(key);
            checkNotNull(loader);
            try {
                // count 保存的是该 segment 中缓存的数量，如果不为 0，则进行获取
                if (count != 0) { // read-volatile
                    /*
                     * don't call getLiveEntry, which would ignore loading values
                     *
                     * 根据 key 和 hash，获取对应的 entry。
                     * 不要直接调用 getLiveEntry 获取 entry，会忽略正在加载的 value
                     */
                    ReferenceEntry<K, V> e = getEntry(key, hash);
                    if (e != null) {
                        long now = map.ticker.read();
                        // getLiveValue 方法在 entry 无效, 过期时会返回 null。如果返回不为空，就是正常命中。
                        V value = getLiveValue(e, now);
                        // 如果获取到了 value
                        if (value != null) {
                            // 记录访问，维护 LRU 队列。
                            recordRead(e, now);
                            // statsCounter 统计信息
                            statsCounter.recordHits(1);
                            // 根据用户是否设置刷新时间(距离上次写入间隔一段时间)，进行刷新或返回旧值
                            return scheduleRefresh(e, key, hash, value, now, loader);
                        }
                        ValueReference<K, V> valueReference = e.getValueReference();
                        // 如果正在加载中，等待加载完成后获取
                        if (valueReference.isLoading()) {
                            // 等待其他线程 load 完成后获取 value
                            return waitForLoadingValue(e, key, valueReference);
                        }
                    }
                }

                /*
                 * at this point e is either null or expired;
                 *
                 * 走到这里说明 value 不存在或者过期，通过 loader 阻塞地加载 value
                 */
                return lockedGetOrLoad(key, hash, loader);
            } catch (ExecutionException ee) {
                Throwable cause = ee.getCause();
                if (cause instanceof Error) {
                    throw new ExecutionError((Error) cause);
                } else if (cause instanceof RuntimeException) {
                    throw new UncheckedExecutionException(cause);
                }
                throw ee;
            } finally {
                postReadCleanup();
            }
        }
```



## Segment#getEntry 方法

com.google.common.cache.LocalCache.Segment#getEntry

根据 key 和 hash，获取对应的 entry。

1. 根据 hash 值定位到桶位置，接下来迭代链表找 key 值也相同的 ReferenceEntry。
2. 发现存在 key 为空的 entry，通过 ReentrantLock#tryLock 方法线程安全地清除。
3. 迭代过程中如果找到了 key 相同的 entry，返回 entry。

```java
        /**
         * 根据 key 和 hash，获取对应的 entry
         *
         * @param key
         * @param hash
         * @return
         */
        @CheckForNull
        ReferenceEntry<K, V> getEntry(Object key, int hash) {
            // 根据 hash 值定位到桶位置，接下来迭代链表找 key 值也相同的 ReferenceEntry
            for (ReferenceEntry<K, V> e = getFirst(hash); e != null; e = e.getNext()) {
                if (e.getHash() != hash) {
                    continue;
                }

                K entryKey = e.getKey();
                // 发现存在 key 为空的 entry，通过 ReentrantLock 线程安全地清除
                if (entryKey == null) {
                    tryDrainReferenceQueues();
                    continue;
                }

                // 找到了 key 相同的 entry，返回 entry
                if (map.keyEquivalence.equivalent(key, entryKey)) {
                    return e;
                }
            }

            // 链表迭代完也没找到 key 对应的 entry，返回 null
            return null;
        }
```



## Segment#getFirst 方法

com.google.common.cache.LocalCache.Segment#getFirst

table 每个元素是一个链表，每个链表是一串连起来的 entry。

这个方法去拿 table 中指定桶位置的开头 entry。

```java
        /**
         * Returns first entry of bin for given hash.
         *
         * table 每个元素是一个链表，每个链表是一串连起来的 entry。
         * 这个方法去拿 table 中指定桶位置的开头 entry。
         */
        ReferenceEntry<K, V> getFirst(int hash) {
            // read this volatile field only once
            AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
            return table.get(hash & (table.length() - 1));
        }
```



## Segment#getLiveValue

com.google.common.cache.LocalCache.Segment#getLiveValue

类似于一个过滤方法，只返回存活有效的 entry。无效的 entry 做清除。

```java
        /**
         * Gets the value from an entry. Returns null if the entry is invalid, partially-collected,
         * loading, or expired.
         *
         * 类似于一个过滤方法，只返回存活有效的 entry。无效的 entry 做清除。
         */
        V getLiveValue(ReferenceEntry<K, V> entry, long now) {
            // key 为 null，返回 null。清除 entry。
            if (entry.getKey() == null) {
                tryDrainReferenceQueues();
                return null;
            }
            V value = entry.getValueReference().get();
            // value 为 null, 返回 null。清除 entry。
            if (value == null) {
                tryDrainReferenceQueues();
                return null;
            }

            // entry 如果过期，返回 null。清除 entry
            if (map.isExpired(entry, now)) {
                tryExpireEntries(now);
                return null;
            }
            return value;
        }
```



## Segment#tryDrainReferenceQueues 方法

com.google.common.cache.LocalCache.Segment#tryDrainReferenceQueues

通过 ReentrantLock 线程安全地清除收集的 entry。一般是清除无效的 entry。

```java
        /**
         * Cleanup collected entries when the lock is available.
         *
         * 通过 ReentrantLock 线程安全地清除收集的 entry。一般是清除无效的 entry。
         */
        void tryDrainReferenceQueues() {
            if (tryLock()) {
                try {
                    drainReferenceQueues();
                } finally {
                    unlock();
                }
            }
        }
```



## Segment#recordRead 方法

com.google.common.cache.LocalCache.Segment#recordRead

记录访问，维护 LRU 队列。

1. 设置 entry 当前访问时间。
2. 维护 LRU 队列。

```java
        /**
         * Records the relative order in which this read was performed by adding {@code entry} to the
         * recency queue. At write-time, or when the queue is full past the threshold, the queue will be
         * drained and the entries therein processed.
         *
         * 记录访问，维护 LRU 队列
         *
         * <p>Note: locked reads should use {@link #recordLockedRead}.
         */
        void recordRead(ReferenceEntry<K, V> entry, long now) {
            if (map.recordsAccess()) {
                // 设置 entry 当前访问时间
                entry.setAccessTime(now);
            }
            // LRU 队列，entry 先前已存在 LRU 队列中，本次调用 add 方法会按照 LRU 的规则调整 entry 的顺序
            recencyQueue.add(entry);
        }
```



## refresh, expire -- 刷新, 过期加载机制

### 三种基于时间的刷新, 过期缓存数据的方式

1. expireAfterAccess: 当缓存项在指定的时间段内没有被访问(读或写)就会过期。
2. expireAfterWrite: 当缓存项在指定的时间段内没有写操作就会被回收。
3. refreshAfterWrite: 当缓存项上一次写操作后间隔多久会刷新。

### expireAfterAccess

1. expireAfterAccess 读操作后会重置过期时间，意味着如果一直读就一直不过期，通常这么做不合适，不选用这种方式。

### expireAfterWrite

1. 考虑时效性，我们可以使用 expireAfterWrite，使每次写操作后指定间隔时间让缓存失效，然后重新加载缓存。Segment 会限制只有 1 个线程能进行加载操作，这样会很好地防止缓存失效的瞬间，大量请求穿透到后端的缓存击穿问题。

2. Segment 在加载操作时加独占锁，在线程安全区域设置一个标志位，然后释放锁，根据标志位判断是否要进行加载。只有一个线程能获取到锁，设置加载标志位，进行加载。其他线程必须阻塞地等待这个加载操作完成才能获取到 value。这样性能会有一些损耗，可预见的，频繁的过期和加载，阻塞等待过程会让性能有较大的损耗。

### refreshAfterWrite

1. 因此我们可以考虑使用 refreshAfterWrite。refreshAfterWrite 的特点是，在 refresh 过程中，Segment 只有 1 个线程做加载操作，其他 get 操作先返回旧值。这样可以有效地减少阻塞耗时，所以 refreshAfterWrite 比 expireAfterWrite 性能好。
2. refreshAfterWrite 缺点是，到达指定时间后，不保证所有的 get 都获取到新值。Segment 没使用额外的线程去做定时清理和加载的功能，而是依赖于调用 get 方法的线程同步处理。在 get 时对比上次更新时间，如超过指定时间则进行加载。所以，如果使用refreshAfterWrite，在吞吐量低的情况下，很长一段时间内没有 get 操作后，发起几个请求触发 get 操作，再次 get 只有一个线程会加载，其他请求线程得到的是一个旧值（这个旧值可能来自于很长时间之前），将会引发问题。

### 组合使用

1. expireAfterWrite 和 refreshAfterWrite 组合使用可以解决各自的缺点。
2. 例如: 控制缓存每 1s 进行 refresh，如果超过 2s 没有访问，则让缓存失效，下次获取时不会得到旧值，而是必须阻塞地等待新值加载，返回新值。

![Screen Shot 2022-10-21 at 10.46.25 AM](https://image-hosting.jellyfishmix.com/20221021105332.png)

1. 这个 get 方法，编号 1 获取存活有效的 entry，即根据 expireAfterAccess 和 expireAfterWrite 判断是否过期。

2. 如果过期，且其他线程没有在加载这个 value，执行编号 3。阻塞地加载 value。
3. 编号 2 在未过期的情况下，根据 refreshAfterWrite 判断是否需要 refresh。



## Segment#lockedGetOrLoad 方法

com.google.common.cache.LocalCache.Segment#lockedGetOrLoad

通过 loader 阻塞地加载 value。

1. createNewEntry 这个标志位表示是否需要加载，会在线程安全区域设置值。退出线程安全区域后根据此值判断是否加载。
2. 获取锁，进入线程安全区域。
3. 根据 key 的 hash 定位桶位置。
4. 遍历链表，链表迭代中定位到 key 对应的 entry。
5. 获得 key 对应的 valueReference。
6. 判断 value 是否正在 loading。如果正在 loading，则不再进行 load 操作（通过设置 createNewEntry 为 false），后续会等待获取新值。
7. 如果需要进行 load 操作，维护一些记录。不用设置 createNewEntry 为 true，因为默认值是 true。
8. 立即重用无效 entry，将此 entry 移除 segment 作用域。
9. 已找到 key 对应的 entry 并处理完毕，停止迭代链表。
10. 如果需要 load，先把 entry 的 valueReference 设置为 loadingValueReference，表示此 value 正在加载中。
11. 释放锁，退出线程安全区域。
12. 如果需要 load，则使用当前线程同步加载 value。进行了加载，说明出现缓存未命中，统计信息。
13. 如果当前线程不需要 load，则调用 Segment#waitForLoadingValue 方法等待其他线程 load 完成后获取 value。

```java
        /**
         * 通过 loader 阻塞地加载 value
         *
         * @param key
         * @param hash
         * @param loader
         * @return
         * @throws ExecutionException
         */
        V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
            ReferenceEntry<K, V> e;
            ValueReference<K, V> valueReference = null;
            LoadingValueReference<K, V> loadingValueReference = null;
            // createNewEntry 这个标志位表示是否需要加载，会在线程安全区域设置值。退出线程安全区域后根据此值判断是否加载。
            boolean createNewEntry = true;

            // 获取锁，进入线程安全区域
            lock();
            try {
                // re-read ticker once inside the lock
                long now = map.ticker.read();
                preWriteCleanup(now);

                int newCount = this.count - 1;
                AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
                // 根据 key 的 hash 定位桶位置
                int index = hash & (table.length() - 1);
                ReferenceEntry<K, V> first = table.get(index);

                // 遍历链表
                for (e = first; e != null; e = e.getNext()) {
                    K entryKey = e.getKey();
                    // 链表迭代中定位到 key 对应的 entry
                    if (e.getHash() == hash
                            && entryKey != null
                            && map.keyEquivalence.equivalent(key, entryKey)) {
                        // 获得 key 对应的 valueReference
                        valueReference = e.getValueReference();
                        // 判断 value 是否正在 loading。如果正在 loading，则不再进行 load 操作（通过设置 createNewEntry 为 false），后续会等待获取新值。
                        if (valueReference.isLoading()) {
                            createNewEntry = false;
                            // 需要进行 load 操作，维护一些记录。不用设置 createNewEntry 为 true，因为默认值是 true
                        } else {
                            V value = valueReference.get();
                            if (value == null) {
                                enqueueNotification(
                                        entryKey, hash, value, valueReference.getWeight(), RemovalCause.COLLECTED);
                                // 如果没有及时 clear up，需要这个二次检查
                            } else if (map.isExpired(e, now)) {
                                // This is a duplicate check, as preWriteCleanup already purged expired
                                // entries, but let's accommodate an incorrect expiration queue.
                                enqueueNotification(
                                        entryKey, hash, value, valueReference.getWeight(), RemovalCause.EXPIRED);
                                // value 已经被其他请求 load 了，统计信息，返回 value。
                            } else {
                                recordLockedRead(e, now);
                                statsCounter.recordHits(1);
                                // we were concurrent with loading; don't consider refresh
                                return value;
                            }

                            /*
                             * immediately reuse invalid entries
                             *
                             * 立即重用无效 entry，将此 entry 移除 segment 作用域
                             */
                            writeQueue.remove(e);
                            accessQueue.remove(e);
                            this.count = newCount; // write-volatile
                        }
                        // 已找到 key 对应的 entry 并处理完毕，停止迭代链表
                        break;
                    }
                }

                // 如果需要 load，先把 entry 的 valueReference 设置为 loadingValueReference，表示此 value 正在加载中
                if (createNewEntry) {
                    loadingValueReference = new LoadingValueReference<>();

                    if (e == null) {
                        e = newEntry(key, hash, first);
                        e.setValueReference(loadingValueReference);
                        table.set(index, e);
                    } else {
                        e.setValueReference(loadingValueReference);
                    }
                }
            } finally {
                // 释放锁，退出线程安全区域
                unlock();
                postWriteCleanup();
            }

            // 如果需要 load，则使用当前线程同步加载 value
            if (createNewEntry) {
                try {
                    // Synchronizes on the entry to allow failing fast when a recursive load is
                    // detected. This may be circumvented when an entry is copied, but will fail fast most
                    // of the time.
                    synchronized (e) {
                        return loadSync(key, hash, loadingValueReference, loader);
                    }
                } finally {
                    // 进行了加载，说明出现缓存未命中，统计信息
                    statsCounter.recordMisses(1);
                }
            } else {
                /*
                 * The entry already exists. Wait for loading.
                 *
                 * 如果当前线程不需要 load，则等待其他线程 load 完成后获取 value
                 */
                return waitForLoadingValue(e, key, valueReference);
            }
        }
```



## Segment#waitForLoadingValue 方法

com.google.common.cache.LocalCache.Segment#waitForLoadingValue

1. 等待其他线程 load 完成后获取 value。主要调用了 valueReference#waitForValue 方法。

2. 对于 LoadingValueReference，waitForValue 方法会以 future.get() 的方式等待获取 value。

```java
        /**
         * 等待其他线程 load 完成后获取 value
         *
         * @param e
         * @param key
         * @param valueReference
         * @return
         * @throws ExecutionException
         */
        V waitForLoadingValue(ReferenceEntry<K, V> e, K key, ValueReference<K, V> valueReference)
                throws ExecutionException {
            if (!valueReference.isLoading()) {
                throw new AssertionError();
            }

            checkState(!Thread.holdsLock(e), "Recursive load of: %s", key);
            // don't consider expiration as we're concurrent with loading
            try {
                // 对于 LoadingValueReference，waitForValue 会以 future.get() 的方式等待获取 value
                V value = valueReference.waitForValue();
                if (value == null) {
                    throw new InvalidCacheLoadException("CacheLoader returned null for key " + key + ".");
                }
                // re-read ticker now that loading has completed
                long now = map.ticker.read();
                recordRead(e, now);
                return value;
            } finally {
                statsCounter.recordMisses(1);
            }
        }
```



## Segment#scheduleRefresh 方法

com.google.common.cache.LocalCache.Segment#scheduleRefresh

根据用户是否设置刷新时间(距离上次写入间隔一段时间)，进行刷新或返回旧值。

1. 判断是否需要 refresh，且当前非 loading 状态。如果是则进行 refresh 操作，并返回新值。
2. 刷新指定的 key 对应的 value。返回值 newValue 有两种情况: 为 null，表示先返回旧值。不为 null 则是加载后的 value。

```java
        /**
         * 根据用户是否设置刷新时间(距离上次写入间隔一段时间)，进行刷新或返回旧值
         *
         * @param entry
         * @param key
         * @param hash
         * @param oldValue
         * @param now
         * @param loader
         * @return
         */
        V scheduleRefresh(
                ReferenceEntry<K, V> entry,
                K key,
                int hash,
                V oldValue,
                long now,
                CacheLoader<? super K, V> loader) {
            // 判断是否需要 refresh，且当前非 loading 状态。如果是则进行 refresh 操作，并返回新值。
            if (map.refreshes()
                    && (now - entry.getWriteTime() > map.refreshNanos)
                    && !entry.getValueReference().isLoading()) {
                // 刷新指定的 key 对应的 value。返回值 newValue 有两种情况: 为 null，表示先返回旧值。不为 null 则是加载后的 value。
                V newValue = refresh(key, hash, loader, true);
                if (newValue != null) {
                    return newValue;
                }
            }
            return oldValue;
        }
```



## Segment#refresh 方法

com.google.common.cache.LocalCache.Segment#refresh

刷新指定的 key 对应的 value。

返回值有两种情况: 为 null，表示先返回旧值。不为 null 则是加载后的 value。

主要逻辑:

1. insertLoadingValueReference 方法用于判断返回旧值，还是需要对 value 进行加载。
2. 走到这里说明当前线程需要做 load 操作，进行 load。
3. 检查 resultFuture，如果已经加载完毕，返回加载后的值。如果尚未加载完毕，返回 null，表示当前线程先返回旧值。

```java
        /**
         * Refreshes the value associated with {@code key}, unless another thread is already doing so.
         * Returns the newly refreshed value associated with {@code key} if it was refreshed inline, or
         * {@code null} if another thread is performing the refresh or if an error occurs during
         * refresh.
         *
         * 刷新指定的 key 对应的 value。
         * 返回值有两种情况: 为 null，表示先返回旧值。不为 null 则是加载后的 value。
         */
        @CanIgnoreReturnValue
        @CheckForNull
        V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
            // insertLoadingValueReference 方法用于判断返回旧值，还是需要对 value 进行加载
            final LoadingValueReference<K, V> loadingValueReference =
                    insertLoadingValueReference(key, hash, checkTime);
            // 这里返回 null，表示当前线程不刷新，返回旧值
            if (loadingValueReference == null) {
                return null;
            }

            // 走到这里说明当前线程需要做 load 操作，进行 load
            ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
            // 检查 resultFuture，如果已经加载完毕，返回加载后的值。如果尚未加载完毕，返回 null，表示当前线程先返回旧值。
            if (result.isDone()) {
                try {
                    return Uninterruptibles.getUninterruptibly(result);
                } catch (Throwable t) {
                    // don't let refresh exceptions propagate; error was already logged
                }
            }
            return null;
        }
```



## Segment#insertLoadingValueReference 方法

com.google.common.cache.LocalCache.Segment#insertLoadingValueReference

1. 判断返回旧值，还是返回一个 LoadingValueReference。
2. 返回 LoadingValueReference 表示需要对 value 进行加载。返回 null 表示返回旧值。
3. 这个方法并不会对 value 进行加载，只是在线程安全区域做一个是否要加载的判断。
4. 可以认为返回值 LoadingValueReference 是加载标记，并发情况下只有一个线程能拿到 LoadingValueReference 加载标记。

主要逻辑：

1. 获取锁，进入线程安全区域。
2. 根据 key 的 hash 定位桶位置。
3. 遍历链表，链表迭代中定位到 key 对应的 entry。
4. 获得 key 对应的 valueReference。
5. 判断 value 是否正在 loading，或写入(加载)间隔时间小于 refresh 阈值。如果满足条件，则返回旧值。如果不满足条件，返回一个 loadingValueReference，表示此线程需要对 value 进行 load。
6. 如果原先 table 中没有对应的 entry，需要新加入一个 entry，并设置 loadingValueReference 表示需要对 value 进行加载。
7. 释放锁，退出线程安全区域。

```java
        /**
         * Returns a newly inserted {@code LoadingValueReference}, or null if the live value reference
         * is already loading.
         *
         * 判断返回旧值，还是返回一个 LoadingValueReference。
         * 返回 LoadingValueReference 表示需要对 value 进行加载。返回 null 表示返回旧值。
         * 这个方法并不会对 value 进行加载，只是在线程安全区域做一个是否要加载的判断。
         * 可以认为返回值 LoadingValueReference 是加载标记，并发情况下只有一个线程能拿到 LoadingValueReference 加载标记。
         */
        @CheckForNull
        LoadingValueReference<K, V> insertLoadingValueReference(
                final K key, final int hash, boolean checkTime) {
            ReferenceEntry<K, V> e = null;
            // 获取锁，进入线程安全区域
            lock();
            try {
                long now = map.ticker.read();
                preWriteCleanup(now);

                AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
                // 根据 key 的 hash 定位桶位置
                int index = hash & (table.length() - 1);
                ReferenceEntry<K, V> first = table.get(index);

                /*
                 * Look for an existing entry.
                 *
                 * 遍历链表
                 */
                for (e = first; e != null; e = e.getNext()) {
                    K entryKey = e.getKey();
                    // 链表迭代中定位到 key 对应的 entry
                    if (e.getHash() == hash
                            && entryKey != null
                            && map.keyEquivalence.equivalent(key, entryKey)) {
                        // We found an existing entry.

                        // 获得 key 对应的 valueReference
                        ValueReference<K, V> valueReference = e.getValueReference();
                        // 判断 value 是否正在 loading，或写入(加载)间隔时间小于 refresh 阈值。如果满足条件，则返回旧值。
                        if (valueReference.isLoading()
                                || (checkTime && (now - e.getWriteTime() < map.refreshNanos))) {
                            /*
                             * refresh is a no-op if loading is pending
                             * if checkTime, we want to check *after* acquiring the lock if refresh still needs
                             * to be scheduled
                             *
                             * 这里返回 null，外层判断返回值为 null，会返回 oldValue
                             */
                            return null;
                        }

                        // continue returning old value while loading
                        ++modCount;
                        // 走到这里，说明原先 table 中没有对应的 entry，需要新加入一个 entry，并设置 loadingValueReference 表示需要对 value 进行加载
                        LoadingValueReference<K, V> loadingValueReference =
                                new LoadingValueReference<>(valueReference);
                        e.setValueReference(loadingValueReference);
                        return loadingValueReference;
                    }
                }

                // 走到这里，说明原先 table 中没有对应的 entry，需要新加入一个 entry，并对 value 进行加载
                ++modCount;
                LoadingValueReference<K, V> loadingValueReference = new LoadingValueReference<>();
                e = newEntry(key, hash, first);
                e.setValueReference(loadingValueReference);
                table.set(index, e);
                return loadingValueReference;
            } finally {
                // 释放锁，退出线程安全区域
                unlock();
                postWriteCleanup();
            }
        }
```



## loadAsync 方法

com.google.common.cache.LocalCache.Segment#loadAsync

使用 loader 加载 value，统计信息并存储加载得到的 value。

请注意，这里同步还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。具体请见 loadFuture 方法。

主要逻辑:

1. 调用 LoadingValueReference#loadFuture 方法，使用 loader 加载 value，返回一个 future。
2. 给 future 设置一个 callback，value 加载完成后触发，统计信息并存储加载得到的 value。

```java
        /**
         * 使用 loader 加载 value，统计信息并存储加载得到的 value。
         * 请注意，这里同步还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。
         * 具体请见 loadFuture 方法: com.google.common.cache.LocalCache.LoadingValueReference#loadFuture(java.lang.Object, com.google.common.cache.CacheLoader)
         *
         * @param key
         * @param hash
         * @param loadingValueReference
         * @param loader
         * @return
         */
        ListenableFuture<V> loadAsync(
                final K key,
                final int hash,
                final LoadingValueReference<K, V> loadingValueReference,
                CacheLoader<? super K, V> loader) {
            /*
             * 使用 loader 加载 value，返回一个 future。
             * 请注意，这里同步还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。具体请见 loadFuture 方法。
             */
            final ListenableFuture<V> loadingFuture = loadingValueReference.loadFuture(key, loader);
            // value 加载完成后触发的 callback
            loadingFuture.addListener(
                    () -> {
                        try {
                            // 统计信息并存储加载得到的 value
                            getAndRecordStats(key, hash, loadingValueReference, loadingFuture);
                        } catch (Throwable t) {
                            logger.log(Level.WARNING, "Exception thrown during refresh", t);
                            loadingValueReference.setException(t);
                        }
                    },
                    directExecutor());
            return loadingFuture;
        }
```



## LoadingValueReference#loadFuture 方法

com.google.common.cache.LocalCache.LoadingValueReference#loadFuture

使用 loader 加载 value，返回一个 future。

请注意，这里同步还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。

主要逻辑：

1. 判断 oldValueReference 是否有值。
   1. 如果 oldValueReference 有值，则可以异步加载，先返回旧值。
   2. 如果 oldValueReference 没有值，则不可以使用异步加载，因为本来就没有可以先返回的旧值，必须使用当前线程同步加载，才有可以返回的值。
2. 使用 loader 加载 value，调用 CacheLoader#reload 方法进行加载。
   1. 这里是同步加载还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。
   2. 默认的 CacheLoader#reload 方法会同步调用我们重写的 CacheLoader#load，所以如果我们传入的 CacheLoader 未重写 reload 方法，默认实现是同步加载。
   3. 我们可以在使用 builder 模式创建 LoadingCache(LocalCache) 时，自己重写 CacheLoader#reload 方法做异步加载。
   4. 也可以使用 CacheLoader.asyncReloading() 方法返回的具有异步加载功能的 CacheLoader 实现类。
3. 返回前对 LoadingValueReference 设置一下加载出来的值。

```java
/**
 * 使用 loader 加载 value，返回一个 future。
 * 请注意，这里同步还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。
 *
 * @param key
 * @param loader
 * @return
 */
public ListenableFuture<V> loadFuture(K key, CacheLoader<? super K, V> loader) {
    try {
        stopwatch.start();
        /*
         * 判断 oldValueReference 是否有值。
         * 如果 oldValueReference 有值，则可以异步加载，先返回旧值。
         * 如果 oldValueReference 没有值，则不可以使用异步加载，因为本来就没有可以先返回的旧值，必须使用当前线程同步加载，才有可以返回的值。
         */
        V previousValue = oldValue.get();
        if (previousValue == null) {
            V newValue = loader.load(key);
            return set(newValue) ? futureValue : Futures.immediateFuture(newValue);
        }
        /*
         * 使用 loader 加载 value，调用 CacheLoader#reload 方法进行加载。
         * 这里是同步加载还是异步加载，主要看传入的 CacheLoader 实现类是否重写了 reload 方法。
         * 默认的 CacheLoader#reload 方法会同步调用我们重写的 CacheLoader#load，所以如果我们传入的 CacheLoader 未重写 reload 方法，默认实现是同步加载。
         * 我们可以在使用 builder 模式创建 LoadingCache(LocalCache) 时，自己重写 CacheLoader#reload 方法做异步加载。
         * 也可以使用 CacheLoader.asyncReloading() 方法返回的具有异步加载功能的 CacheLoader 实现类。
         */
        ListenableFuture<V> newValue = loader.reload(key, previousValue);
        if (newValue == null) {
            return Futures.immediateFuture(null);
        }
        // To avoid a race, make sure the refreshed value is set into loadingValueReference
        // *before* returning newValue from the cache query.
        // 返回前对 LoadingValueReference 设置一下加载出来的值
        return transform(
                newValue,
                newResult -> {
                    LoadingValueReference.this.set(newResult);
                    return newResult;
                },
                directExecutor());
    } catch (Throwable t) {
        ListenableFuture<V> result = setException(t) ? futureValue : fullyFailedFuture(t);
        if (t instanceof InterruptedException) {
            Thread.currentThread().interrupt();
        }
        return result;
    }
}
```

### 默认的 CacheLoader#reload 方法

com.google.common.cache.CacheLoader#reload

默认的 CacheLoader#reload 方法，同步加载。

```java
  @GwtIncompatible // Futures
  public ListenableFuture<V> reload(K key, V oldValue) throws Exception {
    checkNotNull(key);
    checkNotNull(oldValue);
    // 这里可以看到，在同步加载
    return Futures.immediateFuture(load(key));
  }
```

### 异步加载

默认的 CacheLoader#reload 方法会同步调用我们重写的 CacheLoader#load，所以如果我们传入的 CacheLoader 未重写 reload 方法，默认实现是同步加载。

1. 我们可以在使用 builder 模式创建 LoadingCache(LocalCache) 时，自己重写 CacheLoader#reload 方法做异步加载。
2. 也可以使用 CacheLoader.asyncReloading() 方法返回的具有异步加载功能的 CacheLoader 实现类。

使用 builder 模式创建 LoadingCache(LocalCache) 时，自己重写 CacheLoader#reload 方法做异步加载。

```java
  public LoadingCache<String, String> serviceCache =
      CacheBuilder.newBuilder().maximumSize(20000).expireAfterWrite(10, TimeUnit.SECONDS)
          .recordStats().build(new CacheLoader<String, String>() {
        @Override
        public String load(String key) {
          return service.query(key);
        }
      	
      	@Override
      	public ListenableFuture<V> reload(final K key, final V oldValue) {
          ListenableFutureTask<V> task =
              ListenableFutureTask.create(() -> loader.reload(key, oldValue).get());
          executor.execute(task);
          return task;
      	}
      });
```

使用 CacheLoader.asyncReloading() 方法返回的具有异步加载功能的 CacheLoader 实现类举例:

```java
  public LoadingCache<String, String> serviceCache =
      CacheBuilder.newBuilder().maximumSize(20000).expireAfterWrite(10, TimeUnit.SECONDS)
          .recordStats().build(CacheLoader.asyncReloading());
```
