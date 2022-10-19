# ThreadPoolExecutor 源码分析



## 说明

1. 为了方便参考源码，本文在针对源码进行注释时，保留了源码中的全部注释。
2. 默认阅读本文前已经了解了线程池的七大参数和运行机制，由于时间关系这里没有再次贴出。日后有精力的话，或许会考虑在此处整理复习。



## ThreadPoolExecutor 的主要成员属性

```java
    /**
     * Lock held on access to workers set and related bookkeeping.
     * While we could use a concurrent set of some sort, it turns out
     * to be generally preferable to use a lock. Among the reasons is
     * that this serializes interruptIdleWorkers, which avoids
     * unnecessary interrupt storms, especially during shutdown.
     * Otherwise exiting threads would concurrently interrupt those
     * that have not yet interrupted. It also simplifies some of the
     * associated statistics bookkeeping of largestPoolSize etc. We
     * also hold mainLock on shutdown and shutdownNow, for the sake of
     * ensuring workers set is stable while separately checking
     * permission to interrupt and actually interrupting.
     * 
     * mainLock 是线程池锁，在 ThreadPoolExecutor 的很多方法中都有用到，包括 execute() 和 addWorker()，它主要是控制外部线程对线程池对象的安全调用
     */
    private final ReentrantLock mainLock = new ReentrantLock();

	/**
     * The main pool control state, ctl, is an atomic integer packing
     * two conceptual fields
     *   workerCount, indicating the effective number of threads
     *   runState,    indicating whether running, shutting down etc
     *
     * In order to pack them into one int, we limit workerCount to
     * (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2
     * billion) otherwise representable. If this is ever an issue in
     * the future, the variable can be changed to be an AtomicLong,
     * and the shift/mask constants below adjusted. But until the need
     * arises, this code is a bit faster and simpler using an int.
     *
     * The workerCount is the number of workers that have been
     * permitted to start and not permitted to stop.  The value may be
     * transiently different from the actual number of live threads,
     * for example when a ThreadFactory fails to create a thread when
     * asked, and when exiting threads are still performing
     * bookkeeping before terminating. The user-visible pool size is
     * reported as the current size of the workers set.
     *
     * The runState provides the main lifecycle control, taking on values:
     *
     *   RUNNING:  Accept new tasks and process queued tasks
     *   SHUTDOWN: Don't accept new tasks, but process queued tasks
     *   STOP:     Don't accept new tasks, don't process queued tasks,
     *             and interrupt in-progress tasks
     *   TIDYING:  All tasks have terminated, workerCount is zero,
     *             the thread transitioning to state TIDYING
     *             will run the terminated() hook method
     *   TERMINATED: terminated() has completed
     *
     * The numerical order among these values matters, to allow
     * ordered comparisons. The runState monotonically increases over
     * time, but need not hit each state. The transitions are:
     *
     * RUNNING -> SHUTDOWN
     *    On invocation of shutdown(), perhaps implicitly in finalize()
     * (RUNNING or SHUTDOWN) -> STOP
     *    On invocation of shutdownNow()
     * SHUTDOWN -> TIDYING
     *    When both queue and pool are empty
     * STOP -> TIDYING
     *    When pool is empty
     * TIDYING -> TERMINATED
     *    When the terminated() hook method has completed
     *
     * Threads waiting in awaitTermination() will return when the
     * state reaches TERMINATED.
     *
     * Detecting the transition from SHUTDOWN to TIDYING is less
     * straightforward than you'd like because the queue may become
     * empty after non-empty and vice versa during SHUTDOWN state, but
     * we can only terminate if, after seeing that it is empty, we see
     * that workerCount is 0 (which sometimes entails a recheck -- see
     * below).
     *
     * 线程池的控制状态，32 位被分为两部分。高 3 位用于表示五种线程池状态，低 29 位用于线程池持有线程的计数。
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

	/**
     * Tracks largest attained pool size. Accessed only under
     * mainLock.
     * largestPoolSize，该变量记录了线程池在整个生命周期中曾经出现的最大线程个数。线程池创建之后，可以调用 setMaximumPoolSize() 改变运行的最大线程的数目，因此说这里是"曾经"
     */
    private int largestPoolSize;
```



## execute 方法

ThreadPoolExecutor 中最常用的方法了，执行一个 runnable

```java
    /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
        // runnable 判空校验
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        // 获取当前线程池控制状态
        int c = ctl.get();
        // 当前线程数 < 核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 添加一个核心线程。worker 是抽象出的一个线程，用于执行传入的 runnable
            if (addWorker(command, true))
                // 添加 worker 成功，execute 方法返回
                return;
            // 添加 worker 失败，获取当前线程池控制状态
            c = ctl.get();
        }
        // 线程池处于 RUNNING 的状态，尝试向阻塞队列中 offer 此 runnable，添加成功则返回 true
        if (isRunning(c) && workQueue.offer(command)) {
            // 重新获取一下线程池控制状态
            int recheck = ctl.get();
            // 如果线程池不处于正常运行的状态，则从阻塞队列中移除任务
            if (! isRunning(recheck) && remove(command))
                // 从阻塞队列中移除任务成功，触发拒绝策略
                reject(command);
            // 线程池持有的线程数量 == 0
            else if (workerCountOf(recheck) == 0)
                // 添加一个非核心线程，不传递任务
                addWorker(null, false);
        }
        // 线程池不处于正常的运行状态或者阻塞队列已满，则添加一个非核心线程
        else if (!addWorker(command, false))
            // 添加非核心线程失败，触发拒绝策略
            reject(command);
    }
```



## Worker

Worker 截取了主要的成员属性和构造方法。

```java
	/**
     * Class Worker mainly maintains interrupt control state for
     * threads running tasks, along with other minor bookkeeping.
     * This class opportunistically extends AbstractQueuedSynchronizer
     * to simplify acquiring and releasing a lock surrounding each
     * task execution.  This protects against interrupts that are
     * intended to wake up a worker thread waiting for a task from
     * instead interrupting a task being run.  We implement a simple
     * non-reentrant mutual exclusion lock rather than use
     * ReentrantLock because we do not want worker tasks to be able to
     * reacquire the lock when they invoke pool control methods like
     * setCorePoolSize.  Additionally, to suppress interrupts until
     * the thread actually starts running tasks, we initialize lock
     * state to a negative value, and clear it upon start (in
     * runWorker).
     */
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
        /**
         * Thread this worker is running in.  Null if factory fails.
         * 
         * worker 中的线程，当 ThreadFactory 运行错误时可能为空
         * 通过线程池 7 大参数中的 ThreadFactory 创建
         */
        final Thread thread;
        /**
         * Initial task to run.  Possibly null.
         *
         * 初始化时运行的任务，可能为空
         */
        Runnable firstTask;
        
        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker 抑制中断直到 worker 开始运行
            this.firstTask = firstTask;
            // worker 中的线程，是通过线程池 7 大参数中的 ThreadFactory 创建的
            this.thread = getThreadFactory().newThread(this);
        }
    }
```



## addWorker 方法

```java
    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        // 标签跳转，用于 继续下一次/退出 最外层循环。这里会死循环尝试增加线程持有数量
        retry:
        for (;;) {
            // 获取当前线程池的控制状态
            int c = ctl.get();
            // 线程池运行状态
            int rs = runStateOf(c);

            // Check if queue empty only if necessary. 在必要情况下，检查阻塞队列是否为空
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            // 死循环尝试增加线程持有数量
            for (;;) {
                // 当前线程池持有的线程数量
                int wc = workerCountOf(c);
                // 当前线程池持有的线程数量 >= 线程数量溢出上限，几乎不可能，所以为 false
                if (wc >= CAPACITY ||
                    // 如果想 add 一个核心线程，判断 当前线程池持有的线程数量 >= 核心线程数。如果想 add 一个非核心线程，判断 当前线程池持有的线程数量 >= 最大线程数
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    // 达到要添加的线程种类的上限，不可添加
                    return false;
                // 调用 CAS 尝试进行持有线程数 + 1
                if (compareAndIncrementWorkerCount(c))
                    // CAS 添加成功，结束死循环
                    break retry;
                // 因为刚才的 CAS 添加失败，说明其他线程 CAS 修改了线程池持有线程数，重新读取当前线程池的持有线程数
                c = ctl.get();  // Re-read ctl
                // 如果当前线程池的状态发生改变
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop 继续下一次的内层循环
            }
        }

        // 表示 worker 是否启动成功
        boolean workerStarted = false;
        // 表示 worker 是否添加成功
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 实例化一个 worker
            w = new Worker(firstTask);
            // 取出 worker 中的线程
            final Thread t = w.thread;
            // worker 中取出的线程判空校验
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                // 加锁
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    // 获取当前线程池的运行状态
                    int rs = runStateOf(ctl.get());

                    // 如果线程池不处于正常运行的状态
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable 检查线程是否已经已启动
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        // 记录线程池在整个生命周期中曾经出现的最大线程个数
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // worker 添加成功
                        workerAdded = true;
                    }
                } finally {
                    // 解锁
                    mainLock.unlock();
                }
                // worker 成功添加，启动此 worker 的线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // 如果 worker 没有启动成功，则移除此 worker，并维护线程池的持有线程数量
            if (! workerStarted)
                addWorkerFailed(w);
        }
        // 返回 worker 是否启动成功
        return workerStarted;
    }
```



## 如何保证核心线程不会被销毁，keepAlive 机制如何实现的？

这个问题涉及到 runWorker 和 getTask 方法，时间关系未完待续，先写下之前积累的浅显解读。

1. ThreadPoolExecutor -> execute() -> addWorker -> Worker 类的 runWorker -> getTask 方法中：Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
2. 这里 timed 是根据前面判断 当前线程数量是否 > corePoolSize，并结合 allowCoreThreadTimeOut 参数得到的 boolean 类型。可以理解为：
   1. 如果是非核心线程，将会调用带有超时时间的 poll 方法，去存储任务的阻塞队列中尝试获取任务，超时时间就是 设置的 keepAliveTime。
   2. 如果是核心线程，将会调用 take 方法一直阻塞住，等待从任务队列中获取任务，自然核心线程也就无法退出了。