# synchronized 的 monitor 机制



## 前言

1. 本文基于 jdk 8 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## monitor

1. monitor 是 synchronized 中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 class 持有的锁。每一个对象有且仅有一个 monitor。

2. 这也是为什么会看到 synchronized(object) 或 synchronized(MainActivity.class) 这样的写法，这样指明了使用了什么对象或类持有的 monitor。

3. 下图描述了线程和 monitor之间关系，以及线程的状态转换图。

   ![img](https://image-hosting.jellyfishmix.com/20221031231433.png)

4. 如上图，每个 monitor 在每个时刻，只能被一个线程持有，该线程就是 activeThread，其它线程都是 waitingThread，分别在两个队列 entrySet 和 waitSet 里等候。在 entrySet 中等待的线程状态是 waiting for monitor entry，而在 wait set 中等待的线程状态是 in Object.wait()。



## entrySet

1. 先看 entrySet 里面的线程。我们称被 synchronized 包起来的代码段为临界区。

   ```java
   synchronized(object) {
      // ......
   }
   ```
   
2. 当一个线程尝试进入临界区时，有两种可能性:

   1. 该 monitor 不被其它线程持有，entrySet 里面也没有其它等待线程。尝试获取 monitor 的当前线程即成为对应对象或类的 monitor 的 owner，执行临界区的代码。

   2. 该 monitor 被其它线程拥有，尝试获取 monitor 的当前线程在 entrySet 队列中等待。

   3. 第一种情况下，线程将处于 Runnable 的状态。第二种情况下，线程 dump 输出信息会显示处于 waiting for monitor entry。如下:

      ```c++
      "Thread-0" prio=10 tid=0x08222eb0 nid=0x9 waiting for monitor entry [0xf927b000..0xf927bdb8] 
      at testthread.WaitThread.run(WaitThread.java:39) 
      - waiting to lock <0xef63bf08> (a java.lang.Object) 
      - locked <0xef63beb8> (a java.util.ArrayList) 
      at java.lang.Thread.run(Thread.java:595) 
      ```


3. 临界区的机制，是为了保证其内部代码执行的原子性，消除竞态条件。
4. 需要注意的是，临界区在任何时间只允许线程串行通过，这和我们使用多线程提高性能的初衷是相反的。如果在多线程的程序中，大量使用 synchronized，或者不恰当地使用了它，会造成大量线程在临界区的入口等待获取 monitor，造成性能大幅下降。通过 dump 排查问题时，这也是要排查的点。
5. 排查线程阻塞的问题时，cpu 忙则关注 Runnable 的线程，可能正在运行的线程数量太多了，比如线程池 coreSize 设置的太大了。cpu 闲则关注 dump 输出信息中的 waiting for monitor entry，可能大量线程阻塞在获取 monitor。只有获得了 monitor 的线程在占用 cpu 时间片运行，cpu 利用率低。



## waitSet

1. 当线程获得了 monitor，进入临界区之后，如果代码编写调用了 monitor 对应的对象(一般是被 synchronized 修饰的对象)的 wait() 方法，此时放弃 monitor，进入 waitSet 队列。

2. 只有别的线程在该对象上调用了 notify() 或 notifyAll() 方法，waitSet 队列中线程才得到机会去竞争。但是只有一个线程能获得此对象的 monitor，线程状态恢复到 Runnable，其他线程继续待在 waitSet 中。

3. 在 waitSet 中的线程，dump 输出信息中表现为: in Object.wait()。

   ```java
   "Thread-1" prio=10 tid=0x08223250 nid=0xa in Object.wait() [0xef47a000..0xef47aa38] 
    at java.lang.Object.wait(Native Method) 
    - waiting on <0xef63beb8> (a java.util.ArrayList) 
    at java.lang.Object.wait(Object.java:474) 
    at testthread.MyWaitThread.run(MyWaitThread.java:40) 
    - locked <0xef63beb8> (a java.util.ArrayList) 
    at java.lang.Thread.run(Thread.java:595) 
   ```



## 偏向锁

1. ObjectHeader 对象头信息中，有一个标记记录了此对象 monitor 的持有线程，如果尝试获取的线程和此对象 monitor 的持有线程相同，则直接获得 monitor，不需要走锁机制。体现了 synchronized 可重入锁的特征。