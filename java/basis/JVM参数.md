# JVM参数



## 参数

```

Xms 起始内存
Xmx 最大内存
Xmn 新生代内存
Xss 栈大小。 就是创建线程后，分配给每一个线程的内存大小
-XX:NewRatio=n:设置年轻代和年老代的比值。如:为3，表示年轻代与年老代比值为1：3，年轻代占整个年轻代年老代和的1/4
-XX:SurvivorRatio=n:年轻代中Eden区与两个Survivor区的比值。注意Survivor区有两个。如：3，表示Eden：Survivor=3：2，一个Survivor区占整个年轻代的1/5
-XX:MaxPermSize=n:设置持久代大小
收集器设置
-XX:+UseSerialGC:设置串行收集器
-XX:+UseParallelGC:设置并行收集器
-XX:+UseParalledlOldGC:设置并行年老代收集器
-XX:+UseConcMarkSweepGC:设置并发收集器
垃圾回收统计信息
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:filename
并行收集器设置
-XX:ParallelGCThreads=n:设置并行收集器收集时使用的CPU数。并行收集线程数。
-XX:MaxGCPauseMillis=n:设置并行收集最大暂停时间
-XX:GCTimeRatio=n:设置垃圾回收时间占程序运行时间的百分比。公式为1/(1+n)
并发收集器设置
-XX:+CMSIncrementalMode:设置为增量模式。适用于单CPU情况。
-XX:ParallelGCThreads=n:设置并发收集器年轻代收集方式为并行收集时，使用的CPU数。并行收集线程数。
```



## 例题

### 1.

假如某个JAVA进程的JVM参数配置如下：

```
-Xms1G -Xmx2G -Xmn500M -XX:MaxPermSize=64M -XX:+UseConcMarkSweepGC -XX:SurvivorRatio=3
```

请问eden区最终分配的大小是多少？

- A 64M
- B 500M
- C 300M
- D 100M

#### 解析

```
-Xms1G    设置Java堆最小值为1G    
-Xmx2G    设置Java堆最大值为2G
-Xmn500M    设置新生代大小为500M（一个Eden区，两个Survivor区）
-XX:MaxPermSize=64M    设置永久代大小为64M
-XX:+UseConcMarkSweepGC     设置使用CMS收集器
-XX:SurvivorRatio=3    设置Eden区与Survivor区大小的比例
```

本题看新生代大小，新生代为500M，三个区比例为3：1：1，计算出Eden大小为300M