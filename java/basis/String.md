# String



## 基本知识

又是研究过的文章，题目考查的为Java中的字符串常量池和JVM运行时数据区的相关概念。 

### 什么是字符串常量池 

JVM为了减少字符串对象的重复创建，其维护了一个特殊的内存，这段内存被称为字符串常量池或者字符串字面量池 

### 工作原理 

当代码中出现字面量形式创建字符串对象时，JVM首先会对这个字面量进行检查，如果字符串常量池中存在相同内容的字符串数组的引用，则将这个引用返回给堆中的字符串对象的属性（即String类的value属性）。否则新的字符串对象被创建，然后将这个字符串数组引用放入字符串常量池，返回给堆中的字符串对象的属性，最后返回该对象的引用。

### 实现前提（为什么Java中String对象是不可变的？）

字符串常量池实现的前提条件就是Java中String对象是不可变的（本质是字符串常量区的char[]不可变），这样可以安全保证多个String对象共享同一个char[]。如果Java中的String对象可变的话（如果Java中的字符串常量池中的char[]可变的话），一个引用操作改变了字符串常量区char[]的值，那么其他引用了这个char[]的String对象也会受到影响，显然这样是不合理的。

[更详细的关于字符串常量池](http://droidyue.com/blog/2014/12/21/string-literal-pool-in-java/)

### 关于堆和栈 

Java中所有由类实例化的对象和数组都存放在堆内存中，无论是成员变量，局部变量，还是类变量，它们指向的对象都存储在堆内存中。而栈内存用来存储局部变量和方法调用。

[更详细的关于堆和栈的区别](http://droidyue.com/blog/2014/12/07/differences-between-stack-and-heap-in-java/)

#### e.g.

```java
String str = new String("abc");
```

"abc"为字面量对象，其存储在堆内存中。而字符串常量池存储的是char[]：`{'a', 'b', 'c'}`。

### 关于程序寄存器 （程序计数器）

Java中运行时数据区有一个程序寄存器（又称程序计数器），该寄存器为线程私有。Java中的程序计数器用来记录当前线程中正在执行的指令。如果当前正在执行的方法是本地方法，那么此刻程序计数器的值为undefined。

[关于JVM运行时数据区](http://droidyue.com/blog/2014/12/21/java-runtime-data-areas/)

### 成员变量

```java
/** The value is used for character storage. */
private final char[] value;
/** Cache the hash code for the string */
private int hash; // Default to 0
/** use serialVersionUID from JDK 1.0.2 for interoperability */
private static final long serialVersionUID = -6849794470754667710L;
/** Class String is special cased within the Serialization Stream Protocol. */
private static final ObjectStreamField[] serialPersistentFields =
new ObjectStreamField[0];
```

以上是String中的所有成员变量。

### 字符存储

在JDK8时，是将字符存储在char[]中，每个字符将使用两个字节(十六位)。从许多不同的应用程序收集的数据表明字符串是堆使用的主要组成部分，而且，大多数String对象只包含Latin-1字符，这些字符只需要一个字节的存储空间，因此char型的String对象的内部数组有一半空间未使用。

但是从JDK9开始，空间占用方面有了一个优化。String的数据存储格式从

```java
private final char[] value;
```

变成了

```java
private final byte[] value;
```

在JDK9及以后的版本，String源码内部多定义了一个变量

```java
private final byte coder;
```

通过 `coder` 判断使用LATIN1还是UTF16，当字符串都能用LATIN1表示，值就是0，否则就是1。从以下源码可以看出，在处理字符串长度时，如果是char，则长度除以2。

```java
static final boolean COMPACT_STRINGS;
static {
    COMPACT_STRINGS = true;
}
public int length(){
    return value.length >> coder();
}
byte coder(){
    return COMPACT_STRINGS ? coder : UTF16;
}
@Native static final byte LATIN1 = 0;
@Native static final byte UTF16 = 1;
```

### 背景

String作为JDK最核心的数据类型之一，非常有必要专门学习一下，重点关注这4个文件：

- jdk/src/java.base/share/native/libjava/String.c

- jdk/src/java.base/share/classes/java/lang/String.java

- jdk/src/java.base/share/classes/java/lang/StringLatin1.java

- jdk/src/java.base/share/classes/java/lang/StringUTF16.java

### 存储

无论是何种语言的何种实现，String本质上都是字节序列，所有可能的字符加起来就构成了字符集，给字符集中每个字符一个序号就是字符编码，使用最广泛的就是Unicode了，它几乎支持地球上所有常见文字，Unicode有三种最主要的实现，UTF-8，UTF-16还有UTF-32，在web领域，UTF-8已经处于绝对垄断地位。Java 9的String，引入了类似Python str的压缩功能。原理很简单，如果String只包含Latin1字符，1字节存一个字符够用了，如果String含有中文，那么就换一种编码方式存储，用一个变量表示当前的字符集就行了。先看三个类。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {

    /**
     * The value is used for character storage.
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     *
     * Additionally, it is marked with {@link Stable} to trust the contents
     * of the array. No other facility in JDK provides this functionality (yet).
     * {@link Stable} is safe here, because value is never null.
     */
    @Stable
    private final byte[] value;

    /**
     * The identifier of the encoding used to encode the bytes in
     * {@code value}. The supported values in this implementation are
     *
     * LATIN1
     * UTF16
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     */
    private final byte coder;

    /** Cache the hash code for the string */
    private int hash; // Default to 0
}

final class StringLatin1 {
}

final class StringUTF16 {
}
```

字符序列存储在字节数组value中，然后用一个字节的coder表示编码，这是String的基本构成。然后StringLatin1提供一组静态方法，用来处理只含有Latin1字符时的情况。相应的StringUTF16提供另一组静态方法，处理包含Latin1以外字符时的情况。

注意类的定义，三个类都是final，且只有String有public修饰，所以我们作为JDK的用户，只能使用String，而不能使用StringLatin1或者StringUTF16，这两个类不属于API，属于实现细节，我们既不能使用，也不能依赖其内部实现。但是我们应该理解它，顺从它，避免做出违背它的事情来。



## 例题

### 1.

```java
public static void main(String[] args) {
    String a = new String("myString");
    String b = "myString";
    String c = "my" + "String";
    String d = c;
  	// false。a指向堆内存，b指向常量池地址。
    System.out.print(a == b);
  	// false。a指向堆内存，c指向常量池地址。
    System.out.print(a == c);
  	// true。java有常量池优化机制，c和b指向同一个常量池地址。
    System.out.print(b == c);
  	// true。d是c的副本，指向同一个常量池地址。
    System.out.print(b == d);
}
```

### 2.

```java
// "fmn"在常量池中是个不可变量。
String x = "fmn";
// 在堆中创建一个String对象"FMN"，但没有任何引用指向此对象。
x.toUpperCase();
// 在堆中创建一个String对象"Fmn"，y指向此对象。
String y = x.replace('f', 'F');
// 在堆中创建一个String对象"Fmnwxy"，y指向此对象。
y = y + "wxy";
System.out.println(y);
```

### 3.

```java
int x = 20, y = 5;
System.out.println(x + y + "" + (x + y) + y); 
```

#### 解析

1. 不论有什么运算，小括号的优先级都是最高的，先计算小括号中的运算，得到`x + y + "" + 25 + y`。

2. 任何字符与字符串相加都是字符串，但是是有顺序的，字符串前面的按原来的格式相加，字符串后面的都按字符串相加，得到`25 + "" + 25 + 5`。

3. 上面的结果按字符串相加得到`25255`。



## 引用

[探索 Java 中 String 的本质，从 char 说起](https://www.jianshu.com/p/949bd866b24e)

["androidyue"的回答 - 牛客](https://www.nowcoder.com/questionTerminal/4148c53d7e284f19b0f61bc0ada248a8)

[String源码分析(2)--浅析String类 - 梅子酒青木马牛 - 知乎](https://zhuanlan.zhihu.com/p/64537984)

[JDK 9学习笔记 - (2)能屈能伸的String](https://zhuanlan.zhihu.com/p/30584322)

