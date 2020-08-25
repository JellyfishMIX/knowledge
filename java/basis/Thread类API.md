# Thread类API



## sleep 方法

线程的 sleep 方法会使线程休眠指定的时间长度。休眠的意思是，当前逻辑执行到此不再继续执行，而是等待指定的时间。但在这段时间内，该线程持有的 monitor 锁（锁在后面会讲解，这里可以认为对共享资源的独占标志）并不会被放弃。

sleep 方法有两个重载，分别是：

```java
public static native void sleep(long millis) throws InterruptedException;
public static void sleep(long millis, int nanos) throws InterruptedException
```

两者的区别只是一个支持休眠时间到毫秒级，另外一个到纳秒级。但其实第二个并不能真的精确到纳秒级别，我们来看第二个重载方法代码：

```java
public static void sleep(long millis, int nanos)
throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }
    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }
    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }
    sleep(millis);
}
```

可以清楚的看到，最终调用的还是第一个毫秒级别的 sleep 方法。而传入的纳秒会被四舍五入。如果大于 50 万，毫秒 ++，否则纳秒被省略。



## yield 方法

yield 方法我们平时并不常用。yield 单词的意思是让路，在多线程中意味着本线程愿意放弃 CPU 资源，也就是可以让出 CPU 资源。不过这只是给 CPU 一个提示，当 CPU 资源并不紧张时，则会无视 yield 提醒。如果 CPU 没有无视 yield 提醒，那么当前 CPU 会从 RUNNING 变为 RUNNABLE 状态，此时其它等待 CPU 的 RUNNABLE 线程，会去竞争 CPU 资源。刚刚 yield 的线程虽然同为 RUNNABLE 状态，但它不会马上参与竞争获得 CPU 资源。

yield 方法为了提升线程间的交互，避免某个线程长时间过渡霸占 CPU 资源。但 yield 在实际开发中用的比较少，源码的注解也提到这一点：“*It is rarely appropriate to use this method.*”。



## currentThread 方法

静态方法，用于获取当前线程的实例。用法如下：

```java
Thread.currentThread();
```

获取 Thread 的名称：

```java
Thread.currentThread().getName();
```

获取Thread的ID ：

```java
Thread.currentThread().getId();
```



## setPriority 方法

此方法用于设置线程的优先级。每个线程都有自己的优先级数值，当 CPU 资源紧张的时候，优先级高的线程获得 CPU 资源的概率会更大。请注意仅仅是概率会更大，并不意味着就一定能够先于优先级低的获取。这和摇车牌号一个道理，我现在中签概率是标准的 9 倍，但摇中依然摇摇无期。而身边却时不时的出现第一次摇号就中的朋友。如果在 CPU 比较空闲的时候，那么优先级就没有用了，人人都有肉吃，不需要摇号了。

优先级别高可以在大量的执行中有所体现。在大量数据的样本中，优先级高的线程会被选中执行的次数更多。

最后我们看下 setPriority 的源码：

```java
public final void setPriority(int newPriority) {
    ThreadGroup g;
    checkAccess();
    if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
        throw new IllegalArgumentException();
    }
    if((g = getThreadGroup()) != null) {
        if (newPriority > g.getMaxPriority()) {
            newPriority = g.getMaxPriority();
        }
        setPriority0(priority = newPriority);
    }
}
```

Thread 有自己的最小和最大优先级数值，范围在 1-10。如果不在此范围内，则会报错。另外如果设置的 priority 超过了线程所在组的 priority ，那么只能被设置为组的最高 priority 。最后通过调用 native 方法 setPriority0 进行设置。



## interrupt 方法

interrupt 的意思是打断。调用了 interrupt 方法后，线程会怎么样？不知道你的答案是什么。我在第一次学习 interrupt 的时候，第一感觉是让线程中断。其实，并不是这样。inerrupt 方法的作用是让可中断方法，比如让 sleep 中断。也就是说其中断的并不是线程的逻辑，中断的是线程的阻塞。



## join 方法

最后我们再讲解一个重要的方法 join。这个方法功能强大，也很实用。我们用它能够实现并行化处理。比如主线程需要做两件没有相互依赖的事情，那么可以起 A、B 两个线程分别去做。通过调用 A、B 的 join 方法，让主线程 block 住，直到 A、B 线程的工作全部完成，才继续走下去。



## 引用/参考

[ Java并发编程学习宝典 - 07 - 李一鸣 - 慕课网](https://www.imooc.com/read/49/article/940)