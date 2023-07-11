# ThreadPoolExecutor 源码分析



## 说明

1. 本文基于 jdk 8 编写。
2. @author  [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)
4. 默认阅读本文前已经了解了线程池的七大参数和运行机制，由于时间关系这里没有再次贴出。日后有精力的话，或许会考虑在此处整理复习。



## ThreadPoolExecutor 的主要成员属性

```java
    /**
     * mainLock 是线程池锁，在 ThreadPoolExecutor 的很多方法中都有用到，包括 execute() 和 addWorker()，它主要是控制外部线程对线程池对象的安全调用
     */
    private final ReentrantLock mainLock = new ReentrantLock();

	/**
     * 线程池的控制状态，32 位被分为两部分。高 3 位用于表示五种线程池状态，低 29 位用于线程池持有线程的计数。
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

	/**
     * largestPoolSize，该变量记录了线程池在整个生命周期中曾经出现的最大线程个数。线程池创建之后，可以调用 setMaximumPoolSize() 改变运行的最大线程的数目，因此说这里是"曾经"
     */
    private int largestPoolSize;
```



## execute 方法

ThreadPoolExecutor 中最常用的方法了，执行一个 runnable

```java
    public void execute(Runnable command) {
        // runnable 判空校验
        if (command == null)
            throw new NullPointerException();
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



## ThreadPoolExecutor 使用一个线程反复执行 Runnable 的方式

java.util.concurrent.ThreadPoolExecutor#runWorker

1. while 循环里反复调用 getTask 从阻塞队列里获得 Runnable, 然后执行 Runnable#run 方法，调用传入 runnable 时重写的 run 方法。

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // while 循环里反复调用 getTask 从阻塞队列里获得 Runnable, 然后执行 Runnable#run 方法
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行 Runnable#run 方法, 调用传入 runnable 时重写的 run 方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```



## 如何保证核心线程不会被销毁，keepAlive 机制如何实现的？

这个问题涉及到 runWorker 和 getTask 方法，时间关系未完待续，先写下之前积累的浅显解读。

1. ThreadPoolExecutor -> execute() -> addWorker -> Worker 类的 runWorker -> getTask 方法中：Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
2. 这里 timed 是根据前面判断 当前线程数量是否 > corePoolSize，并结合 allowCoreThreadTimeOut 参数得到的 boolean 类型。可以理解为：
   1. 如果是非核心线程，将会调用带有超时时间的 poll 方法，去存储任务的阻塞队列中尝试获取任务，超时时间就是 设置的 keepAliveTime。
   2. 如果是核心线程，将会调用 take 方法一直阻塞住，等待从任务队列中获取任务，自然核心线程也就无法退出了。

