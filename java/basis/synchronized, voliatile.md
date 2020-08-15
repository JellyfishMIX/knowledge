# synchronized, voliatile



## synchronized修饰静态方法和非静态方法的区别

### 1. Synchronized修饰非静态方法，实际上是对调用该方法的对象加锁，俗称“对象锁”。

Java中每个对象都有一个锁，并且是唯一的。假设分配的一个对象空间，里面有多个方法，相当于空间里面有多个小房间，如果我们把所有的小房间都加锁，因为这个对象只有一把钥匙，因此同一时间只能有一个人打开一个小房间，然后用完了还回去，再由JVM 去分配下一个获得钥匙的人。

#### 情况1：同一个对象在两个线程中分别访问该对象的两个同步方法。

结果：会产生互斥。

解释：因为锁针对的是对象，当对象调用一个synchronized方法时，其他同步方法需要等待其执行结束并释放锁后才能执行。

#### 情况2：不同对象在两个线程中调用同一个同步方法。

结果：不会产生互斥。

解释：因为是两个对象，锁针对的是对象，并不是方法，所以可以并发执行，不会互斥。形象的来说就是因为我们每个线程在调用方法的时候都是new 一个对象，那么就会出现两个空间，两把钥匙。

### 2. Synchronized修饰静态方法，实际上是对该类对象加锁，俗称“类锁”。

#### 情况1：用类直接在两个线程中调用两个不同的同步方法

结果：会产生互斥。

解释：因为对静态对象加锁实际上对类（.class）加锁，类对象只有一个，可以理解为任何时候都只有一个空间，里面有N个房间，一把锁，因此房间（同步方法）之间一定是互斥的。

注：上述情况和用单例模式声明一个对象来调用非静态方法的情况是一样的，因为永远就只有这一个对象。所以访问同步方法之间一定是互斥的。

#### 情况2：用一个类的静态对象在两个线程中调用静态方法或非静态方法

结果：会产生互斥。

解释：因为是一个对象调用，同上。

#### 情况3：一个对象在两个线程中分别调用一个静态同步方法和一个非静态同步方法

结果：不会产生互斥。

解释：因为虽然是一个对象调用，但是两个方法的锁类型不同，调用的静态方法实际上是类对象在调用，即这两个方法产生的并不是同一个对象锁，因此不会互斥，会并发执行。



## volatile

### 定义

由volatile定义的变量其特殊性在于：

一个线程对变量的写一定对之后对这个变量的读的线程可见。

换言之

一个线程对volatile变量的读一定能看见它之前最后一个线程对这个变量的写。

### 机理

volatile意味着可见性，在讲解volatile的机理前，我先给下面的这个例子：

```Java
package com.cielo.main;

/**
 * Created by 63289 on 2017/3/31.
 */
class MyThread extends Thread {
    private boolean isRunning = true;
    public boolean isRunning() {
        return isRunning;
    }
    public void setRunning(boolean isRunning) {
        this.isRunning = isRunning;
    }
    @Override
    public void run() {
        System.out.println("进入到run方法中了");
        while (isRunning == true) {
        }
        System.out.println("线程执行完成了");
    }
}
public class RunThread{
    public static void main(String[] args) {
        try {
            MyThread thread = new MyThread();
            thread.start();
            Thread.sleep(1000);
            thread.setRunning(false);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

在这个例子中，主线程启动了子线程，子线程成功进入run方法，输出”进入到run方法中”，只有由于isRunning==true，无限循环。此时，sleep一秒后的主线程想要改变isRunning的值，它将isRunning变量读取到它的内存空间进行修改后，写入主内存，但由于子线程一直在私有栈中读取isRunning变量，没有在主内存中读取isRunning变量，因此不会退出循环。

如果我们把isRunning赋值行改为：

private volatile boolean isRunning = true;
将其用volatile修饰，则强制该变量从主内存中读取。

这样我们也就明白了volatile的实现机理，即：

1. 当一个线程要使用volatile变量时，它会直接从主内存中读取，而不使用自己工作内存中的副本。
2. 当一个线程对一个volatile变量写时，它会将变量的值刷新到共享内存(主内存)中。

### 特性：不会被重排序

从Java内存模型一篇中，我们简单了解了重排序，这里不会被重排序主要指语句重排序。

我们考虑到下面这个例子，有A,B两个线程

线程A：加载配置文件，将配置元素初始化，之后标识初始化成功。

```Java
Map configOptions ;
char[] configText;

volatile boolean initialized = false;

//线程A首先从文件中读取配置信息,调用process...处理配置信息,处理完成了将initialized 设置为true
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfig(configText, configOptions);//负责将配置信息configOptions 成功初始化
initialized = true;
线程B：等待初始化标识为true，之后开始工作。

while(!initialized)
{
    sleep();
}

//使用配置信息干活
doSomethingWithConfig();
```

很简单的一个例子，在编译器中，如果进行重排序，则会有将initialized=true这一行先执行的可能，如果这件事发生的话，线程B就会先运行，进而使用了没有加载配置文件的Object。而如果initialized变量使用了volatile修饰，则编译器不会将该变量的相关代码进行重排序。（当然，这里的例子只是为了直观，实际情况编译器的重排序会更加复杂）

### 非原子性

使用volatile时，我们要清楚，volatile是非原子性的。

原子性即是指，对于一个操作，其操作的内容只有全部执行/全不执行两个状态，不存在中间态。而volatile并不能锁定某组操作，防止其他线程的干扰，即没有规定原子性，因而volatile是非原子性的。或者说，volatile是非线程安全的。

综上，如果我们想要使用一个原子性的修饰符来控制操作，即在操作变量时锁定变量，我们就需要另一个修饰词synchronized。



## synchronized与voliatile区别

1. 使用：voliatile 用于修饰变量，synchronized可以修饰对象，类，方法，代码块，语句。

2. 原子性：voliatile只保证变量的可见性，不能用于同步变量，即不保证原子性，多线程并发访问voliatile修饰的变量时也不会产生阻塞。synchronized是原子性的，只有锁定了变量的线程才能进入临界区，从而保证临界区的所有语句全部执行。多线程并发访问sychronized修饰的变量会产生阻塞。

3. 机理：

   当线程对volatile变量读时，会把工作内存中值置为无效。当线程对sychronized变量读时，会在该线程锁定变量时把工作内存中值置为无效。

   当线程对voliatile变量写时，会把值刷新到主内存中。当线程对sychronized变量写时，会在变量解锁时把值刷新到主内存中。



## synchronized特点

1. 无论synchronized加在方法上还是对象上，其修饰的都是对象，而不是方法或者某个代码块代码语句。
2. 每个对象只有一个锁与之相关联。
3. 实现同步需要很大的系统开销来做控制，不要做无谓的锁定。



## synchronized的作用域

synchronized的作用域只有两种。实际上，synchronized直接作用于内存中的一个内存块，因此，可以通过锁定内存块来锁定一个实例变量或者锁定一个静态区域。

1. 某个对象实例内

synchronized aMethod(){}可以防止多个线程同时访问这个对象的synchronized方法，如果对象有多个synchronized方法，则只要一个线程访问了任何一个synchronized方法，其他线程不能同时访问任何一个该对象的synchronized方法(synchronized作用于对象，且每个对象只有一个锁)。

显然，不同对象的synchronized方法则不会互相影响(synchronized作用于对象)。

1. 某个类的范围

又或者说作用于静态方法/静态代码块。synchronized static aMethod(){}防止多个线程同时访问这个类中的synchronized static方法，它可以对类的所有实例对象起作用。



## synchronized可重入性

### 可重入的条件

- 不在函数内使用静态或全局数据。
- 不返回静态或全局数据，所有数据都由函数的调用者提供。
- 使用本地数据（工作内存），或者通过制作全局数据的本地拷贝来保护全局数据。
- 不调用不可重入函数。

### 可重入与线程安全

一般而言，可重入的函数一定是线程安全的，反之则不一定成立。在不加锁的前提下，如果一个函数用到了全局或静态变量，那么它不是线程安全的，也不是可重入的。如果我们加以改进，对全局变量的访问加锁，此时它是线程安全的但不是可重入的，因为通常的枷锁方式是针对不同线程的访问（如Java的synchronized），当同一个线程多次访问就会出现问题。只有当函数满足可重入的四条条件时，才是可重入的。

### synchronized可重入性

回到引言里的问题，如果一个获取锁的线程调用其它的synchronized修饰的方法，会发生什么？

从设计上讲，当一个线程请求一个由其他线程持有的对象锁时，该线程会阻塞。当线程请求自己持有的对象锁时，如果该线程是重入锁，请求就会成功，否则阻塞。

我们回来看synchronized，synchronized拥有强制原子性的内部锁机制，是一个可重入锁。因此，在一个线程使用synchronized方法时调用该对象另一个synchronized方法，即一个线程得到一个对象锁后再次请求该对象锁，是永远可以拿到锁的。

在Java内部，同一个线程调用自己类中其他synchronized方法/块时不会阻碍该线程的执行，同一个线程对同一个对象锁是可重入的，同一个线程可以获取同一把锁多次，也就是可以多次重入。原因是Java中线程获得对象锁的操作是以线程为单位的，而不是以调用为单位的。

### synchronized可重入锁的实现

之前谈到过，每个锁关联一个线程持有者和一个计数器。当计数器为0时表示该锁没有被任何线程持有，那么任何线程都都可能获得该锁而调用相应方法。当一个线程请求成功后，JVM会记下持有锁的线程，并将计数器计为1。此时其他线程请求该锁，则必须等待。而该持有锁的线程如果再次请求这个锁，就可以再次拿到这个锁，同时计数器会递增。当线程退出一个synchronized方法/块时，计数器会递减，如果计数器为0则释放该锁。



## 引用/参考

[使用synchronized修饰静态方法和非静态方法有什么区别 - riemann_ - CSDN](https://blog.csdn.net/riemann_/article/details/99245845)

[Java多线程：线程间通信之volatile与sychronized - CieloSun的博客](https://www.cnblogs.com/cielosun/p/6650161.html)