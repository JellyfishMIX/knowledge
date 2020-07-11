# Stream



## IO流

![138512_1427527478646_1](https://image-hosting.jellyfishmix.com/20200621100835.png)



##  流操作

Java的流操作分为字节流和字符流两种。 字节流与字符流主要的区别是他们的处理方式 。

- 字节流是最基本的，所有的`InputStream`和`OutputStream`及其子类都是字节流，主要用在处理二进制数据，它是按字节来处理的。
- 实际中很多的数据是文本，又提出了字符流的概念，它是按虚拟机的`encode`来处理，也就是要进行字符集的转化。

这两个之间通过 `InputStreamReader`，`OutputStreamWriter`来关联，实际上是通过`byte[]`和`String`来关联。 

在实际开发中出现的汉字问题实际上都是在字符流和字节流之间转化不统一而造成的。

字符流和字节流都有缓冲流。



### 字节流 -> 字符流
字节流转化为字符流，实际上就是`byte[]`转化为`String`时，

```java
public String(byte bytes[], String charsetName)
```

有一个关键的参数字符集编码，通常我们都省略了，那系统就用操作系统的`lang`。



### 字符流 -> 字节流

字符流转化为字节流，实际上是`String`转化为`byte[]`时：

```java
byte[] String.getBytes(String charsetName)
```

java.io中还出现了许多其他的流，按主要是为了提高性能和使用方便，如`BufferedInputStream`，`PipedInputStream`等。



## 缓冲流

- 不带缓冲的操作，每读一个字节就要写入一个字节，由于涉及磁盘的IO操作相比内存的操作要慢很多，所以不带缓冲的流效率很低。
- 带缓冲的流，可以一次读很多字节，但不向磁盘中写入，只是先放到内存里。等凑够了缓冲区大小的时候一次性写入磁盘，这种方式可以减少磁盘操作次数，提高速度。

e.g.

```
InputStream
|__FilterInputStream
        |__BufferedInputStream
```



## 节点流和处理流

按照流是否直接与特定的地方（如磁盘、内存、设备等）相连，分为节点流和处理流两类。 

- 节点流：可以从或向一个特定的地方（节点）读写数据，如`FileReader`。 
- 处理流：是对一个已存在的流的连接和封装，通过所封装的流的功能调用实现数据读写。如`BufferedReader`处理流的构造方法总是要带一个其他的流对象做参数。一个流对象经过其他流的多次包装，称为流的链接。

### JAVA常用的节点流：

- 文件 `FileInputStream`, `FileOutputStrean`,  `FileReader`, `FileWriter`文件进行处理的节点流。 
- 字符串 `StringReader`, `StringWriter`对字符串进行处理的节点流。 
- 数组 `ByteArrayInputStream`, `ByteArrayOutputStreamCharArrayReader`, `CharArrayWriter`对数组进行处理的节点流（对应的不再是文件，而是内存中的一个数组）。 
- 管道 `PipedInputStream`, `PipedOutputStream`, `PipedReaderPipedWriter`对管道进行处理的节点流。

### 常用处理流（关闭处理流使用关闭里面的节点流）

- 缓冲流：`BufferedInputStream`, `BufferedOutputStream`, `BufferedReader`, `BufferedWriter`增加缓冲功能，避免频繁读写硬盘。 

- 转换流：`InputStreamReader`, `OutputStreamReader`实现字节流和字符流之间的转换。 
- 数据流 `DataInputStream`, `DataOutputStream`等提供将基础数据类型写入到文件中，或者读取出来。

### 流的关闭顺序

- 一般情况下是：先打开的后关闭，后打开的先关闭。

- 另一种情况：看依赖关系，如果流a依赖流b，应该先关闭流a，再关闭流b。例如，处理流a依赖节点流b，应该先关闭处理流a，再关闭节点流b。

- 可以只手动关闭处理流，不用手动关闭节点流。处理流关闭的时候，会调用其处理的节点流的关闭方法。



## 引用

["学不会Java"的回答-牛客网](https://www.nowcoder.com/test/question/done?tid=34202769&qid=373223#summary)

["无情的AC机器"的回答-牛客网](https://www.nowcoder.com/test/question/done?tid=34266953&qid=372736#summary)