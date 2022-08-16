# Thread & Runnable



## Java中实现多线程的两种基本方式

- 继承Thread类，重写run()。调用thread.start();

- 实现Runnable接口，重写run()。调用thread.start();

第二种方式是更好的实践，原因如下：

- java语言中只能单继承，通过实现接口的方式，可以让实现类去继承其它类。而直接继承thread就不能再继承其它类了。

- 线程控制逻辑在Thread类中，业务运行逻辑在Runnable实现类中。解耦更为彻底。

- 实现Runnable的实例，可以被多个线程共享并执行。而实现thread是做不到这一点的。



为什么程序中调用的是Thread的start()方法，而不是run()方法？



## Thread start方法源代码分析

我们先看Thread类start方法源代码，如下：

```java
public synchronized void start() {
    if (threadStatus != 0)
        throw new IllegalThreadStateException();
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
        }
    }
}
```

这段代码足够简单，简单到没什么内容。主要逻辑如下：

1. 检查线程的状态，是否可以启动；
2. 把线程加入到线程group中；
3. 调用了start0()方法。

可以看到Start方法中最终调用的是start0方法，并不是run方法。那么我们再看start0方法源代码：

```java
private native void start0();
```

什么也没有，因为start0是一个native方法，也称为JNI（Java Native Interface）方法。JNI方法是java和其它语言交互的方式。同样也是java代码和虚拟机交互的方式，虚拟机就是由C++和汇编所编写。

由于start0是一个native方法，所以后面的执行会进入到JVM中。那么run方法到底是何时被调用的呢？这里似乎找不到答案了。

难道我们错过了什么？回过头来我们再看看Start方法的注解。其实读源代码的时候，要先读注解，否则直接进入代码逻辑，容易陷进去，出不来。原来答案就在start方法的注解里，我们可以看到：

```java
* Causes this thread to begin execution; the Java Virtual Machine
* calls the <code>run</code> method of this thread.
* <p>
* The result is that two threads are running concurrently: the
* current thread (which returns from the call to the
* <code>start</code> method) and the other thread (which executes its
* <code>run</code> method).
* <p>
* It is never legal to start a thread more than once.
* In particular, a thread may not be restarted once it has completed
* execution.
```

最关键一句*the Java Virtual Machine calls the run method of this thread。*由此我们可以推断出整个执行流程如下：
![图片描述](https://image-hosting.jellyfishmix.com/20200916091659.jpg)

start方法调用了start0方法，start0方法在JVM中，start0中的逻辑会调用run方法。



## Thread Run方法分析

对于上面提出的问题，我们先从Thread的构造函数入手。原因是Runnable的实现对象通过构造函数传入Thread。

```java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
```

可以看到Runnable实现作为target对象传递进来。再次调用了init方法，init 方法有多个重载，最终调用的是如下方法：

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals)
```

此方法里有一行代码：

```java
this.target = target;
```

原来target是Thread的成员变量：

```java
/* What will be run. */
private Runnable target;
```

此时，Thread的target被设置为你实现业务逻辑的Runnable实现。

我们再看下run方法的代码：

```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```

看到这里是不是已经很清楚了，当你传入了target，则会执行target的run方法。也就是执行你实现业务逻辑的方法。

整体执行流程如下：

![图片描述](https://image-hosting.jellyfishmix.com/20200916092413.jpg)

如果你是通过继承Thread，重写run方法的方式实现多线程。那么在第三步执行的就是你重写的run方法。

我们回过头看看Thread类的定义：

```java
public class Thread implements Runnable
```

原来Thread也实现了Runnable接口。怪不得Thread类的run方法上有@Override注解。所以继承thread类实现多线程，其实也相当于是实现runnable接口的run方法。只不过此时，不需要再传入一个Thread类去启动。它自己已具备了thread的功能，自己就可以运转起来。既然Thread类也实现了Runnable接口，那么thread子类对象是不是也可以传入另外的thread对象，让其执行自己的run方法呢？答案是可行的，可以亲手试一下。



## Thread.interrupt()

首先，一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。
所以，Thread.stop, Thread.suspend, Thread.resume 都已经被废弃了。
而 Thread.interrupt 的作用其实也不是中断线程，而是「通知线程应该中断了」，
具体到底中断还是继续运行，应该由被通知的线程自己处理。

具体来说，当对一个线程，调用 interrupt() 时，
① 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
② 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。

interrupt() 并不能真正的中断线程，需要被调用的线程自己进行配合才行。

也就是说，一个线程如果有被中断的需求，那么就可以这样做：
① 在正常运行任务时，经常检查本线程的中断标志位，如果被设置了中断标志就自行停止线程。
② 在调用阻塞方法时正确处理InterruptedException异常。（例如，catch异常后就结束线程。）

```java
Thread thread = new Thread(() -> {
    while (!Thread.interrupted()) {
        // do more work.
    }
});
thread.start();

// 一段时间以后
thread.interrupt();
```



## 引用/参考

[ Java并发编程学习宝典 - 05 - 李一鸣 - 慕课网](https://www.imooc.com/read/49/article/940)

[Java里一个线程调用了Thread.interrupt()到底意味着什么？- Intopass - 知乎](https://www.zhihu.com/question/41048032/answer/89431513)