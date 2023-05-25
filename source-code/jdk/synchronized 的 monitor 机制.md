# synchronized 的 monitor 机制



## 前言

1. 本文基于 jdk 8 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## monitor

1. monitor 是 synchronized 中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者 class 持有的锁。每一个对象有且仅有一个 monitor。

2. 这也是为什么会看到 synchronized(object) 或 synchronized(MainActivity.class) 这样的写法，这样指明了使用了什么对象或类持有的 monitor。

3. 下图描述了线程和 monitor之间关系，以及线程的状态转换图。

   ![img](https://image-hosting.jellyfishmix.com/20230411200828.png)

4. 如上图，每个 monitor 在每个时刻，只能被一个线程持有，该线程就是 activeThread，其它线程都是 waitingThread，分别在两个队列 entryList 和 waitSet 里等候。在 entryList 中等待的线程状态是 waiting for monitor entry，而在 wait set 中等待的线程状态是 in Object.wait()。



## entryList

1. 先看 entryList 里面的线程。我们称被 synchronized 包起来的代码段为临界区。

   ```java
   synchronized(object) {
      // ......
   }
   ```

2. 当一个线程尝试进入临界区时，有两种可能性:

   1. 该 monitor 不被其它线程持有，entryList 里面也没有其它等待线程。尝试获取 monitor 的当前线程即成为对应对象或类的 monitor 的 owner，执行临界区的代码。

   2. 该 monitor 被其它线程拥有，尝试获取 monitor 的当前线程在 entryList 队列中等待并休眠。

   3. 第一种情况下，线程将处于 Runnable 的状态。第二种情况下，线程 dump 输出信息会显示处于 waiting for monitor entry。如下:

      ```bash
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

1. 当线程获得了 monitor，进入临界区之后，如果代码编写调用了 monitor 对应的对象(一般是被 synchronized 修饰的对象)的 wait() 方法，此时放弃 monitor，进入 waitSet 队列并休眠。

2. 只有别的线程在该对象上调用了 notify() 或 notifyAll() 方法，waitSet 队列中的线程才进入 entryList 中等待竞争。但是只有一个线程能获得此对象的 monitor，线程状态恢复到 Runnable，其他线程继续在 entryList 中等待并休眠。

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



## synchronized 的锁升级机制(偏向锁, 轻量级锁, 重量级锁)

entryList 的机制被称为重量级锁，性能较差。jdk 1.6 从 JVM 层⾯对 synchronized 进行了⼤优化，引入了偏向锁, 轻量级锁和锁升级机制，提高了 synchronized 的性能。

### 偏向锁

1. ObjectHeader 对象头信息中，有一个标记记录了此对象 monitor 的持有线程，如果尝试获取的线程和此对象 monitor 的持有线程相同，则直接获得 monitor，不需要走锁机制。体现了 synchronized 可重入锁的特征。

### 轻量级锁

1. 当一个线程尝试进入临界区时，如果获取偏转锁失败，则进行轻量级锁的机制，CAS 自旋尝试抢锁。
2. 这个设计是为了提高锁的性能，如果在 CAS 自旋期间抢到了锁，则可以减少线程休眠和唤醒的开销。

### 重量级锁

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230419130103.png)

1. 自旋 n 次(默认为10，可以通过 JVM 参数 PreBlockSpin 调整) 如果仍未抢到，则升级为重量级锁。
1. 线程进入 contentionList, 一个链表，每次通过 CAS 在队头添加 node, 而出队(取线程)操作发生在队尾。
1. contentionList 会被线程并发访问，为了降低对 contentionList 的并发争用，而引入 entryList。
1. owner 线程在 unlock 时会从 contentionList 中迁移线程到 entryList，并会指定 entryList 中某个线程(一般是第一个)为 OnDeck 线程(候选线程)。owner 线程并不是把锁传递给 OnDeck 线程，只是把竞争锁的权利交给 OnDeck，OnDeck 线程需要竞争抢锁。这里体现了 synchronized 是非公平锁。
1. 线程进入 entryList 后。entryList 的机制如本文前部分所述。



## 参考文章

synchronized重量级锁原理1 -- 别给我加香菜 -- 掘金 https://juejin.cn/post/7180527422235181115

Synchronized实现原理和锁优化 -- zhifeng687 -- CSDN https://blog.csdn.net/qq_26222859/article/details/53786134

谈谈Java中的锁机制 -- 我超爱JAVA的 -- CSDN https://blog.csdn.net/Dev_Hugh/article/details/106577862