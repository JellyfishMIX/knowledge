# ThreadLocal 源码分析



## 说明

1. 本文基于 jdk 8 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 应用场景

ThreadLocal 主要用来存储当前上下文的变量信息，它可以保证存储进去的信息，只能被当前的线程读取到，线程之间 ThreadLocal 隔离。



## ThreadLocal demo

demo 展示了 ThreadLocal 怎么用。

```java
package Thread;
 
import java.util.Random;
 
public class ThreadLocalDemo {
	public static class MyRunnable implements Runnable {
		private ThreadLocal<Integer> threadLocal = new ThreadLocal<>();
        
		@Override
		public void run() {
			threadLocal.set(new Random().nextInt(100));
			try{
				Thread.sleep(2000);
			} catch(InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println(Thread.currentThread() + ":" + threadLocal.get());
		}
	}
    
	public static void main(String[] args) {
		System.out.println("start");
		MyRunnable runnable = new MyRunnable();
		Thread thread1 = new Thread(runnable);
		Thread thread2 = new Thread(runnable);
		thread1.start();
		thread2.start();
	}
}
```



## 关键属性

### ThreadLocalMap

ThreadLocalMap 是 ThreadLocal 的内部类，一个线程有且仅有一个属于自己的 ThreadLocalMap。ThreadLocal 作为 key 存储在 ThreadLocalMap 中。通过 ThreadLocalMap，保证存储进去的信息，只能被当前的线程读取到，线程之间 ThreadLocal 隔离。

后面会对 ThreadLocalMap 做详细的介绍。

### ThreadLocalMap table

存储 entry 的 table，长度必须是 2 的倍数。ThreadLocalMap 使用线性探测法解决 hash 冲突(内部只维护 Entry 数组，没有链表)。

我们可以参考 HashMap 的概念，把 table 称为"桶"，table 上的一个格就是一个桶位置。

```java
static class ThreadLocalMap {
        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         *
         * 存储 entry 的 table。长度必须是 2 的倍数
         * ThreadLocalMap 使用线性探测法解决 hash 冲突(内部只维护 Entry 数组，没有链表)。
         */
        private Entry[] table;
}
```

### ThreadLocalMap Entry

entry 是 ThreadLocalMap 的一个内部类，负责存储 key-value 的元素，存储在 table 中。我们注意到 key 被存储为弱引用(WeakReference)，后面解释原因。

```java
static class ThreadLocalMap {
		/**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         *
         * entry 的 key 是弱引用的。弱引用的对象，发生 gc 时，如果未被任何强类型的对象引用，那么，该弱引用的对象就会被 gc 回收。
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
}
```

### ThreadLocalMap 的初始容量

INITIAL_CAPACITY 是 ThreadLocalMap 的初始容量，必须是 2 的倍数

```java
static class ThreadLocalMap {
		/**
         * The initial capacity -- MUST be a power of two.
         *
         * ThreadLocalMap 的初始容量，必须是 2 的倍数
         */
        private static final int INITIAL_CAPACITY = 16;
}
```

### ThreadLocalMap 的加载因子

ThreadLocalMap 的加载因子是固定的，写死在了 setThreshold 方法中，加载因子是 2 / 3。

```java
static class ThreadLocalMap {
		/**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         *
         * 加载因子为 2 / 3
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
}
```



## ThreadLocal set 方法

```java
public class ThreadLocal<T> {
	/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        // threadLocalMap 存在，往里面 set
        if (map != null)
            map.set(this, value);
        else
            // threadLocalMap 不存在，则初始化
            createMap(t, value);
    }
}
```

可以看到 set 方法中出现了一个 ThreadLocalMap，ThreadLocalMap 是负责存储本线程 ThreadLocal 对象的。 ThreadLocalMap 通过 getMap 方法获得。当前 ThreadLocal 当作 key，调用 ThreadLocalMap 的 set 方法把 key- value 设置进去。如果当前线程的 threadLocalMap 不存在，会调用 createMap 方法进行初始化。



## ThreadLocal getMap 方法

```java
public class ThreadLocal<T> {
	/**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * 从指定的 Thread 中拿到 ThreadLocalMap。这也说明 ThreadLocalMap 是每个线程各自持有一个的。
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
}
```

从指定的 Thread 中拿到 ThreadLocalMap。这也说明 ThreadLocalMap 是每个线程各自持有一个的。

```java
public class Thread implements Runnable {
	/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class.
     *
     * 存储当前线程的 ThreadLocal。这个属性由 ThreadLocal 进行维护。
     */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

threadLocals 是 Thread 的一个成员属性，存储当前线程的 ThreadLocal。这个属性由 ThreadLocal 进行维护。

如果 getMap 方法返回为空，说明指定的 Thread 还没有初始化 ThreadLocalMap，需要调用 createMap 方法进行初始化。



## ThreadLocal createMap 方法

```java
public class ThreadLocal<T> {
	/**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * 为指定线程初始化 ThreadLocalMap
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map  初始化后添加的第一个元素
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

为指定线程初始化 ThreadLocalMap，第一个参数 t 是指定的线程，第二个参数 firstValue 是 ThreadLocalMap 初始化后添加的第一个元素。

ThreadLocalMap 是 ThreadLocal 的内部类，构造方法如下：

### ThreadLocalMap 的构造方法

```java
static class ThreadLocalMap {
		/**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         * 
         * 初始化 ThreadLocalMap，firstValue 指初始化后存入的第一个元素
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            // 初始容量 16
            table = new Entry[INITIAL_CAPACITY];
            // 类似于 HashMap 的桶位置定位公式，key 的 hash 值 & (table 长度 - 1)
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            // 在桶位置上构建一个 Entry，存储 key 和 value
            table[i] = new Entry(firstKey, firstValue);
            // 维护 ThreadLocalMap 的元素计数
            size = 1;
            // 设置扩容阈值
            setThreshold(INITIAL_CAPACITY);
        }
}
```



## ThreadLocalMap 的 set 方法

ThreadLocal 在拿到当前线程的 ThreadLocalMap 后，会调用 ThreadLocalMap 的 set 方法，下面是 set 方法的介绍：

```java
static class ThreadLocalMap {
		/**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            // 存储 entry 的 table
            Entry[] tab = table;
            // table 的长度
            int len = tab.length;
            // 桶位置定位公式，key 的 hash 值 & (table 长度 - 1)
            int i = key.threadLocalHashCode & (len-1);

            /*
             * 循环尝试插入 Entry，使用线性探测法解决 hash 冲突(只维护 Entry 数组，没有链表)
             * 这里不会因为 table 中没有空桶位置而导致死循环，因为加载因子保证了扩容阈值比 table 长度小，一定存在空桶位置。
             */
            for (Entry e = tab[i];
                // 循环结束条件：当 e 为 null 时，循环结束
                e != null;
                // 维护下一次迭代中 e 的值，进行桶位置重定位
                e = tab[i = nextIndex(i, len)]) {
                // 获取当前桶位置上的 key
                ThreadLocal<?> k = e.get();

                // 如果桶位置上的 key 与要插入的 key 相同，则覆盖之前的 value
                if (k == key) {
                    e.value = value;
                    return;
                }

                /*
                 * 如果当前桶位置不为空（存在 entry），但是 entry 的 key 为空，说明是一个 stale entry，使用新的 key-value 替换它
                 * 同时此方法中也有逻辑：清除 ThreadLocalMap 中的 stale entry，并进行 rehash
                 */
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            // 走到这里说明找到了新增 key 对应的桶位置，根据 key-value 构建一个 Entry
            tab[i] = new Entry(key, value);
            // 维护 ThreadLocalMap 的元素计数
            int sz = ++size;
            /*
             * 清除 ThreadLocalMap 中的 stale entry，并进行 rehash
             * 如果桶位置上的 key 为空，则说明之前的弱引用 key 已经在 gc 中被回收。
             * 这个 entry 没用了，称这样的 entry 为 stale。清除 ThreadLocalMap 中的 stale entry
             * ThreadLocalMap 使用线性探测法解决 hash 冲突(内部只维护 Entry 数组，没有链表)
             * 因此清除 ThreadLocalMap 中的 stale entry 后，会进行 rehash。防止 table 的当前位置为 null 后，有 hash 冲突的 Entry 访问不到的问题。
             */
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                // 当ThreadLocalMap 的元素计数达到扩容阈值时，进行扩容（虽然名字是 rehash，但是里面有 resize 扩容逻辑）
                rehash();
        }
}
```



### ThreadLocalMap 桶位置重定位

ThreadLocalMap 使用线性探测法解决 hash 冲突(内部只维护 Entry 数组，没有链表)。而桶位置重定位的 nextIndex 方法为：

```java
static class ThreadLocalMap {
		/**
         * Increment i modulo len.
         *
         * 根据 i 和 len，获得下一个桶位置
         * 当 (i + 1 < len) 时，返回 i 的下一个桶位置偏移量，即返回 i + 1
         * 当 (i + 1 >= len) 时，返回 0。意思是，超过 0 也没关系，再从 0 开始用线性探测法找
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
}
```



## ThreadLocal get 方法

```java
public class ThreadLocal<T> {
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        // threadLocalMap 是否存在
        if (map != null) {
            // threadLocalMap 存在，调用 ThreadLocalMap 的 getEntry 方法，获得 key 对应的 entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            // entry 不为空，拿 value
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        /*
         * 给 key 对应的 entry 设置一个初始值
         * 进入 setInitialValue 方法有两种情况：
         * 1. ThreadLocalMap 不存在，或者 2. key 定位到的桶位置 entry 为 null
         */
        return setInitialValue();
    }
}
```

get 方法逻辑清晰，代码中已注释，值得关注的地方是当 1. ThreadLocalMap 不存在，或者 2. key 定位到的桶位置 entry 为 null 时调用的方法 setInitialValue()



### ThreadLocal setInitialValue 方法

```java
public class ThreadLocal<T> {
	/**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * 给 key 对应的 entry 设置一个初始值，即给 key 设置一个 entry 占位
     * 进入 setInitialValue 方法有两种情况：
     * 1. ThreadLocalMap 不存在，或者 2. key 定位到的桶位置 entry 为 null
     * ThreadLocalMap 使用线性探测法解决 hash 冲突，是靠判断桶位置是否有 entry，来判断是否与其他 key 发生了 hash 碰撞
     * 如果不给 key 设置一个 entry 占位，下次 set 别的 key 可能定位到此桶位置，发现没有 entry，以为没有发生 hash 碰撞，于是设置了 entry。
     * 这样会出现两个 key 都定位到了一个桶位置的情况，都可以拿到这个桶位置上的 entry。
     *
     * @return the initial value
     */
    private T setInitialValue() {
        // initialValue 方法返回一个 null
        T value = initialValue();
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取当前线程的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
            // 给 key 设置一个 entry 占位
            map.set(this, value);
        // ThreadLocalMap 不存在，初始化
        else
            createMap(t, value);
        // 返回设置的占位 entry 的 value，相当于返回 null
        return value;
    }
}
```

这个方法值得关注的问题：为什么 ThreadLocal set 方法发现当前线程的 ThreadLocalMap 不存在，直接 createMap 初始化。而 ThreadLocal get 方法发现当前线程的 ThreadLocalMap 不存在，或 key 定位到的桶位置 entry 为 null，会调用 setInitialValue 设置一个 entry 占位呢？

因为 ThreadLocalMap 使用线性探测法解决 hash 冲突，是靠判断桶位置是否有 entry，来判断是否与其他 key 发生了 hash 碰撞。如果不给 key 设置一个 entry 占位，下次 set 别的 key 可能定位到此桶位置，发现没有 entry，以为没有发生 hash 碰撞，于是设置了 entry。这样会出现两个 key 都定位到了一个桶位置的情况，都可以拿到这个桶位置上的 entry。



## 未完待续

1. ThreadLocalMap 的 set 方法中，ThreadLocal 获取 key 的 hash 值：threadLocalHashCode -> nextHashCode() -> AtomicInteger 的 nextHashCode.getAndAdd(HASH_INCREMENT)
2. ThreadLocal 中的 Entry 为什么被设计成弱引用？
3. ThreadLocal 在线程池中的污染问题

以上问题时间精力不足暂未分析，未完待续...
