1. jdk 1.7 及以前 ConcurrentHashMap 采用了分段锁，每个线程使用各自的段 Segament 进行读写操作，不会影响到其他线程的读写操作。jdk 1.8 之后也是线程安全的，请看如下分析。

2. jdk 1.7 及以前 ConcurrentHashMap 采用了分段锁。从 jdk 1.8 及以后，ConcurrentHashMap 抛弃了分段锁的设计，改用 数组 + 链表/红黑树 的结构。3. put 操作时 cas + synchronized 实现线程安全。put 操作拿 key 的 hashcode 进行 hashcode & (length - 1) 运算，定位桶位置。当定位到桶位置为空时 cas 尝试写入。如果定位到桶位置不为空，则使用 synchronized 控制链表和树的 add 操作。

4. get 操作时，拿 key 的 hashcode 进行 hashcode & (length - 1) 运算，定位桶位置。如果桶位置为空，则返回null。如果桶位置为单纯的值，则直接读出 value。如果桶位置为树，则使用树的查询方式，在树中未查询到key 对应的 value，则返回空。如果桶位置为链表，则遍历链表 读出 value，如果遍历链表结束扔未找到 key 对应的 value，则返回空。