# LinkedBlockingQueue 的源码分析



## offer(E e) 方法[非阻塞]

offer 方法呈现的效果: offer(E e): 表示如果可能的话,将 e 添加到 BlockingQueue 里,即如果 BlockingQueue 可以容纳，则返回 true，否则返回 false。本方法不阻塞当前执行方法。

```java
	/**
     * Current number of elements
     * 队列的当前元素个数
     */
    private final AtomicInteger count = new AtomicInteger();

	/**
     * The capacity bound, or Integer.MAX_VALUE if none
     * 队列的容量上限
     */
    private final int capacity;

	/**
     * Lock held by put, offer, etc
     * 写锁，put, offer 方法共用这把锁。可以理解为添加元素时用的写锁
     */
    private final ReentrantLock putLock = new ReentrantLock();

	/** 
     * Wait queue for waiting puts
     * 等待添加元素的线程
     */
    private final Condition notFull = putLock.newCondition();

	/**
     * Linked list node class
     * 链表实现的队列，单个存储单元
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }

	/**
     * Inserts the specified element at the tail of this queue if it is
     * possible to do so immediately without exceeding the queue's capacity,
     * returning {@code true} upon success and {@code false} if this queue
     * is full.
     * When using a capacity-restricted queue, this method is generally
     * preferable to method {@link BlockingQueue#add add}, which can fail to
     * insert an element only by throwing an exception.
     *
     * @throws NullPointerException if the specified element is null
     */
	public boolean offer(E e) {
        // 判空校验
        if (e == null) throw new NullPointerException();
        // 队列的当前元素个数
        // 1. 把成员变量 count 赋值给了局部变量 count 2. 局部变量用 final 修饰了
        final AtomicInteger count = this.count;
        // 如果当前队列的元素个数已达容量上限，则直接返回 false
        if (count.get() == capacity)
            return false;
        // 表示添加元素后，队列的当前元素个数
        int c = -1;
        // node，新建一个存储单元
        Node<E> node = new Node<E>(e);
        // 当前的写锁，put, offer 方法共用这把锁。可以理解为添加元素时用的写锁
        final ReentrantLock putLock = this.putLock;
        // 加锁，进入线程安全区
        putLock.lock();
        try {
            // 在加锁后的线程安全区，判断队列中是否还有剩余容量
            if (count.get() < capacity) {
                // 入队列
                enqueue(node);
                // 队列容量 + 1
                c = count.getAndIncrement();
                // 如果添加元素后，队列仍然没有满，则唤醒一个等待添加元素的线程
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            // 解锁
            putLock.unlock();
        }
        // 如果添加元素后，队列非空，则唤醒一个等待拿取(take)的线程
        if (c == 0)
            signalNotEmpty();
        // 表示是否添加成功
        return c >= 0;
    }
```

额外的注释：

```java
// 队列的当前元素个数
final AtomicInteger count = this.count;
```

这里值得关注的问题有两处:

1. 把成员变量 count 赋值给了局部变量 count
2. 局部变量用 final 修饰了

回答 1. 这里把成员变量 count 赋值给了局部变量 count。访问成员变量，每次都需要读内存。但是局部变量可以放入 cpu 缓存，这样先把成员变量赋值给局部变量，读 cpu 缓存的时间会快于读内存。不过这种把成员变量先读成局部变量的方式，属于极端优化。正常写业务不用这么写，还可能增加心智负担。性能差异不大，不如考虑开发效率，降低心智负担。

回答 2.

![image-20220715144828474](https://image-hosting.jellyfishmix.com/20220715144828.png)

![image-20220715144852239](https://image-hosting.jellyfishmix.com/20220715144852.png)

我们先来看两张图的对比，可以发现对局部变量赋值的操作，加与不加 final，编译后的字节码是相同的。字节码里没有任何东西能体现出局部变量的 final 与否，Class 文件里除字节码（Code 属性）外的辅助数据结构也没有记录任何体现 final 的信息。既然带不带 final 的局部变量在编译到 Class 文件后都一样了，其访问效率必然一样高，JVM 不可能有办法知道什么局部变量原本是用 final 修饰来声明的。

（题外话: 有一个例外，编译常量时，final 常量折叠机制: [JVM对于声明为final的局部变量（local var）做了哪些性能优化？ - RednaxelaFX的回答 - 知乎]( https://www.zhihu.com/question/21762917/answer/19239387)）

猜测，截图中，jdk 源码里作者加 final 修饰，应该是为了避免自己手滑在后面改写了局部变量里最初读到的值，而加上 final 来让编译器（javac 之类）检查。



## put(E e) 方法[阻塞]

put 方法呈现的效果：put(E e) 把 e 加到 BlockingQueue 里，如果 BlockingQueue 没有空间，则调用此方法的线程被阻塞直到BlockingQueue 里面有空间再继续。

```java
	/**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary for space to become available.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        // 判空校验
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        // 表示添加元素后，队列的当前元素个数
        int c = -1;
        // node，新建一个存储单元
        Node<E> node = new Node<E>(e);
        // 当前的写锁，put, offer 方法共用这把锁。可以理解为添加元素时用的写锁
        final ReentrantLock putLock = this.putLock;
        // 队列的当前元素个数
        final AtomicInteger count = this.count;
        // 后面 put 方法可能会阻塞住当前线程，因此加一个可中断的锁。进入线程安全区
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            // 如果当前队列的元素个数已达容量上限，则当前线程阻塞住，直到被信号唤醒或线程被中断
            while (count.get() == capacity) {
                notFull.await();
            }
            // 入队列
            enqueue(node);
            // 队列容量 + 1
            c = count.getAndIncrement();
            // 如果添加元素后，队列仍然没有满，则唤醒一个等待添加元素的线程
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            // 解锁
            putLock.unlock();
        }
        // 如果添加元素后，队列非空，则唤醒一个等待拿取(take)的线程
        if (c == 0)
            signalNotEmpty();
    }
```



## poll() 方法

```java
	/**
     * Lock held by take, poll, etc
     * 读锁，take, poll 方法共用这把锁。可以理解为拿取元素时用的读锁
     */
    private final ReentrantLock takeLock = new ReentrantLock();
```

