# JVM safe point 安全点和 counted loop 可数循环



## 前言

本文基于 jdk 8 写作。



## 问题引出

```java
public class JvmCountedLoop {
    private static AtomicInteger num = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = () -> {
            for (int i = 0; i < 100_000_000; i++) {
                num.getAndAdd(1);
            }
            // safe point
            System.out.println(Thread.currentThread().getName() + "执行结束!");
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();

        long startTime = System.currentTimeMillis();
        Thread.sleep(1000);
        // safe point
        long endTime = System.currentTimeMillis();
        System.out.println("num = " + num);
        System.out.format("consume time: %d ms\n", endTime - startTime);
    }
}
```

输出结果：

![image-20220908220335657](https://image-hosting.jellyfishmix.com/20220908220335.png)

这段代码按照平常的思维，consume time 应该输出 1000 ms。但实际并没有按照期望输出 1000 ms，而是 3270 ms。

```java
		long startTime = System.currentTimeMillis();
        Thread.sleep(1000);
        // safe point
        long endTime = System.currentTimeMillis();
```



## JVM safe point 安全点

JVM 会在一些适当的位置插入安全点，当程序执行到安全点时，称为进入安全点。

Thread.sleep() 方法是一个 native 方法，在从 native 方法返回后，JVM 会在当前线程设置一个安全点，让线程进入这个安全点。

关于安全点的介绍可以看这篇文章：https://www.javatt.com/p/47329

简单地说，每一个线程进入安全点后，会检测一个标志位，来决定自己是否暂停执行。

请注意：

1. 在本文中我们称其为 STW 标志位，请注意，这不是官方称呼。
2. 我们为了方便理解，暂且称 STW 标志位为 true 时，线程会暂停执行。

当一个线程在 safe point 由于 STW 标志位为 true 而暂停时，需要等待一次 GC 执行后才能离开安全点。

图中主线程在进入 safe point 后，发现 STW 标志位为 true（为什么 STW 标志位为 true 后面再分析），阻塞在安全点，需要等待一次 GC 执行后才能离开安全点。这也是为什么 consume time 多花了两秒钟。

为什么这一次 GC 等待的时间（STW）很久呢？因为 GC 触发的条件是所有线程都进入安全点后才会触发。



## JVM counted loop 机制

除了主线程外，另外两个线程在 for 循环里兜圈呢，没有进入到安全点。另外两个线程在 for 循环中不会进入安全点的原因是，hotspot 的 counted loop 机制。

《深入理解JVM虚拟机(第三版)》5.2.8 小节：

![img](https://image-hosting.jellyfishmix.com/20220908222652.png)

简单总结一下，当 for 循环中索引值是 int 类型，称为 counted loop。hotsopt 不会在这个循环里放置 safe point。要等待 counted loop 结束后才会放置 safe point。

```java
		Runnable runnable = () -> {
            for (int i = 0; i < 100_000_000; i++) {
                num.getAndAdd(1);
            }
            // safe point
            System.out.println(Thread.currentThread().getName() + "执行结束!");
        };
```

### 破坏 counted loop

我们把 for 循环中的 int 改为 long，可以破坏 counted loop，变成 uncounted loop：

```java
public class JvmCountedLoop {
    private static AtomicInteger num = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = () -> {
            for (long i = 0; i < 100_000_000; i++) {
                num.getAndAdd(1);
            }
            // safe point
            System.out.println(Thread.currentThread().getName() + "执行结束!");
        };

        Thread t1 = new Thread(runnable);
        Thread t2 = new Thread(runnable);
        t1.start();
        t2.start();

        long startTime = System.currentTimeMillis();
        Thread.sleep(1000);
        // safe point
        long endTime = System.currentTimeMillis();
        System.out.println("num = " + num);
        System.out.format("consume time: %d ms\n", endTime - startTime);
    }
}
```

输出结果：

![image-20220909001509582](https://image-hosting.jellyfishmix.com/20220909001509.png)

可以看到，本次 consume time 耗时符合预期，1004 ms

4 ms 偏差解释一下：

线程苏醒后只是切换为了 Runnable 状态，还是要等操作系统调度，操作系统给分配 cpu 时间片才能运行。

比如操作系统就是过了 4 ms 才给分配 cpu 时间片，那就会有 4 ms 偏差。

在截图中，我们把 for 循环中的 int 改为 long，破坏 counted loop，变成 uncounted loop。此时 JVM 允许在循环中插入 safe point。

这样主线程在到达 safe point 后，发现 STW 标志位为 true，阻塞在安全点。很快其他线程也会到达 safe point，发生一次 GC，然后各线程离开安全点继续执行。

### 演示一个对比实验，把 sleep 睡觉时间调小

我们试试，把 sleep 睡觉的时间调小一些会怎么样。输出结果如下：

![image-20220909003547936](https://image-hosting.jellyfishmix.com/20220909003547.png)

consume time 10 ms。主线程没有被阻塞在安全点。

主线程从 sleep native 方法返回后进入 safe point，检查 STW 标志位，发现不为 true，离开安全点继续执行。

对实验结果解释一下原因：

应该是在 10 ms - 1000 ms 之间的某个时刻，JVM 做出了判断，要把 STW 标志位置为 true。

如果是 10 ms，sleep 结束，进入了安全点，但是此时 JVM 还没做出置 STW 标志位为 true 的决定。

因此主线程不会阻塞住。

### 演示另一个对比实验，把 for 循环次数调大

我们试试，把 for 循环次数调大一些会怎么样。输出结果如下：

![image-20220909003709395](https://image-hosting.jellyfishmix.com/20220909003709.png)

consume time 6564 ms。同样符合之前的结论，主线程被阻塞在安全点，等待 GC 执行后才能离开安全点。

而且由于 counted loop 花费的时间更长了，导致主线程要在安全点等更久，其他线程才能完成 counted loop，进入安全点。所有线程都到达安全点，主线程才等到 GC 触发，然后离开安全点继续执行。



## JVM 何时决定把 STW 标志位置为 true

线程进入 safe point 后，不一定会阻塞，他只是去检测 STW 标志位，来决定自己是否要暂停。

STW 标志位是由 JVM 进行维护的，JVM 会在运行时做出判断，是否要将 STW 标志位置为 true。

JVM 何时决定置 STW 标志位为 true，准确判断标准目前未知。猜测：当 JVM 结合运行时环境，预估稍后一段时间内，是否会产生大量可回收对象，来决定是否要把 STW 标志位置为 true。比如存在较长时间的循环时，JVM 可能会认为将产生大量可回收对象，决定把 STW 标志位置为 true。来让各线程在下一次进入安全点时，进行一次 GC。

当然这是猜想，准确的原因由于笔者水平不够，暂不知道。



## 线上环境 counted loop 导致的问题

关于 counted loop 会导致其他线程进入安全点后等待 GC 时间过长，这种担忧在线上环境真的存在。

这篇文章是小米云技术官方发表的：HBase实战：[记一次Safepoint导致长时间STW的踩坑之旅](https://juejin.cn/post/6844903878765314061)

![image-20220909002609107](https://image-hosting.jellyfishmix.com/20220909002609.png)

小米的工程师亲身实战过，总结的经验，解决方案有两种：
1. UseCountLoopSafepoints JVM 参数，允许 counted loop 循环中插入安全点。
2. for 循环 index 类型从 int 改成 long，破坏 counted loop，变成 uncounted loop，来允许在循环中插入安全点。

