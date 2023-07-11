# javaNIO -- ByteBuffer 原理机制



## 说明

1. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 概述

1. ByteBuffer 可以理解为是一个 byte 数组，用于读取与写入。
2. ByteBuffer 通过一些精巧的属性和方法, 更高效地使用内存空间。
3. java NIO 中有 8 种缓冲区: ByteBuffer, CharBuffer, DoubleBuffer, FloatBuffer, IntBuffer, LongBuffer, ShortBuffer, MappedByteBuffer。其中最常用的是 ByteBuffer。



## ByteBuffer 属性

1. capacity: 表示缓冲区数组的最大容量。
2. position: 表示当前指针的位置(下一个要操作的数据元素的位置)
3. limit: 表示当前数组最大的使用量, 即有效位置的 EOF 位置(缓冲区数组中不可操作的下一个元素的位置), limit <= capacity
4. mark: 用于记录某个时刻 position 的位置快照, 默认是 -1。这个属性不常用，为了便于理解忽略即可。

### 不同场景下属性的作用

1. 写模式下, position 是写入位置, limit 等于容量。

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230603235507.png)

2. 调用 flip 方法后,  position 切换为读取位置,  limit 切换为读取限制。

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604000558.png)

3. 读取了 4 个字节之后的状态如下:

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604000612.png)

4. clear 动作发生后:

   ![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604001332.png)



## ByteBuffer 方法

### clear

切换为写模式。

将参数设置为 position=0, limit=capacity, mark=-1，类似于初始化，但不会清除 byte 数组的内容(即 clear 只是把 position 移到位置 0，并没有真正清空数据)。

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604015501.png)

### flip

切换为读模式。

将参数设置为 limit=position, position=0, mark=-1, 翻转, 即将未翻转之前 0 到 position 之间的数据是翻转后的 position(即0) 到 limit 之间的这块区域，翻转将缓冲区的状态由写变为读。

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604015531.png)

### rewind

将参数设置为 position=0, mark=-1, limit 的值不变(注意：指针指向0)。这样在读完一次后，把 position 重定向为 0, 即可从头开始读取。

### reset

把 position 设置为 mark 的值，相当于之前通过 mark 记录了一个 position 位置快照，现在回退到快照位置。

### remaining

return limit - position，即返回 limit 和 position 之间的相对位置差。表示未读取完的剩余数据量。

### hasRemaining

return position < limit, 表示是否有未读取完的剩余数据。

### compact

切换为写模式。

将 position 与 limit 之间的内容移动到 0 到 (limit - position) 之间的区域, 之前 0 到 position 间的数据被覆盖。position 的值变为 limit - position，limit 的值变为 capacity。如果之前 position == limit, 此时执行 compact 操作相当于 clear 操作。

![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20230604015405.png)

### get

可以理解为 getByte, 从 byteBuffer 里读取 1 个 byte, 以 byte 类型返回。同时 position += 1。

### get(int index)

可以理解为 getByte(int index), 从 byteBuffer 里读取指定 position 位置的 1 个 byte, 以 byte 类型返回。但 position 位置不变。

### getInt

从 byteBuffer 里读取 4 个 byte, 以 int 类型返回。同时 position += 4。

byteBuffer 里存储的都是 byte 类型的数据，调用方指定要 int 类型的返回值, 那就拿出来 4 个 byte, 以 int 类型返回。

### getLong

从 byteBuffer 里读取 8 个 byte, 以 long 类型返回。同时 position += 4。

byteBuffer 里存储的都是 byte 类型的数据，调用方指定要 long 类型的返回值, 那就拿出来 8 个 byte, 以 long 类型返回。



## 使用方法

1. ByteBuffer 初始状态是写模式, 使用 IO 流即可写入数据, 例如: channel.read(byteBuffer)
2. 调用 flip 方法, 可以切换为读取模式。
3. 调用 clear 或者 compact 方法切换至写模式。
4. ByteBuffer 读取方法有很多, 最常用的是 get 方法。



## 拓展阅读

[Java NIO 的数据运输大队：Buffer 与 Channel 详解 - 肖恩Sean - 掘金](https://juejin.cn/book/7207773992437710882/section/7213544651960090659)

[java.nio.ByteBuffer中的flip()、rewind()、compact()等方法的使用和区别 - 峰的季节 - 博客园](https://www.cnblogs.com/chdf/p/11466522.html)
