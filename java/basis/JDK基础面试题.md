# JDK基础面试题



## 前言

来看看某招聘网站上的常见的 java 工程师的 JD（见下图），绝大部分职位都会要求 **Java 基础扎实**。

![图片描述](https://image-hosting.jellyfishmix.com/20200929141829.jpg)

何为 java 基础扎实？广义地讲，包括 **Java 基础能力、集合类、JVM、多线程和并发、IO 和队列**等，即对应专栏的第二个模块 **Java 基础技术**；而狭义地讲，则对应其中的第一个主题 **Java 基础能力** ，又可分为三部分：

**第一部分是 JDK 相关的基础能力**，主要体现在对 java 基础类库的了解；

**第二部分是面向对象基础能力**，主要体现对 java 编程语言和面向对象思想的了解；

**第三部分是 java 基础的进阶能力**，主要体现对 java 语言的高阶用法的了解（如反射等）。

这三部分将分三个章节讲述。



## 1. JDK 基础知识结构及面试题目分析



### 1.1 JDK 基础知识结构

JDK1.8 中 java 包下面的一级包有 14 个，其中需要我们熟练掌握的有四个（下图中标红部分）。
![图片描述](https://image-hosting.jellyfishmix.com/20200929141839.jpg)



### 2.2 面试考察点

Java 基础部分主要考察的是候选人对 java 基础知识的掌握能力。

对新人而言，面试官需要考察其基础是否扎实，考察兼具深度与广度；对一定经验的人而言，面试官需要考察其是否已远离一线编程，重点在深度，回答时更要全面。

所以尽管都是些基础性的问题，但是难度却并不一定小，需要仔细分析。一旦回答错误较多，那面试考察项中的 “基础不牢” 怕是跑不掉了。

> 基础部分的面试题考察方向基本一致，所以后面两章节不再对此进行展开说明。



## 2. 典型面试例题及思路分析

**问题 1："你常用的 JDK 类有哪些？请说出 5 个。"**

**参考答案：**

String、StringBuffer、Integer、ArrayList、HashMap、Date、Object（选择其中候选人熟悉的 5 个）

**点评：**

这是一道典型的开放题，通常具有如下作用：

（1) 题目开放简单，营造融洽的面试氛围，引导候选人进入状态；

（2) 每个人的答案不一样，便于下一步个性化的考察。

注意第 2 点，这也是这道题虽然开放简单却万不可掉以轻心的原因。因为很可能下一个问题就会基于你的答案进一步发问，所以题干虽然问的是你常用的 JDK 类，但也不可随口乱说，而一定要从精心准备的几个熟悉的类中挑选，一般 5 个类可以由以下几个展开：String (确实最常用）、ArrayList 和 HashMap（集合类、几乎必问）、其他可用的类（比如说类似 Integer 之类）。

> 题目变种及小 Tips：
>
> 如果题干中换种问法：“你常用的 java 类有哪些？”，那么除了 JDK 中的类，还可以选一些流行框架中的类，比如说 fastjson 中的 JSONObject，apache 中的 BeanUtils 等等。选择的标准有两个：一是自己确实熟悉，二是方便展开，"吸引" 面试官继续发问。

**问题 2："String、StringBuilder、StringBuffer 的区别是什么？"**

**参考答案：**

1、可变性。String 不可变，StringBuilder 与 StringBuffer 是可变的。

- String 类中使用只读字符数组保存字符串，private final char value []，所以是不可变的（Java 9 中底层把 char 数组换成了 byte 数组，占用更少的空间)。
- StringBuilder 与 StringBuffer 都继承自 AbstractStringBuilder 类，在 AbstractStringBuilder 中也是使用字符数组保存字符串，char [] value，这两种对象都是可变的。

2、线程安全性。String 和 StringBuffer 是线程安全的，StringBuilder 是非线程安全的。

- String 线程安全是因为其对象是不可变的，StringBuffer 线程安全是因为对方法加了同步锁或者对调用的方法加了同步锁。
- StringBuilder 并没有对方法进行加同步锁，所以是非线程安全的。

3、性能。

- String 的性能较差，因为每次对 String 类型进行改变的时候，都会生成一个新的 String 对象，然后将指针指向新的 String 对象。
- 而 StringBuffer/StringBuilder 性能更高，是因为每次都是对对象本身进行操作，而不是生成新的对象并改变对象引用。一般情况下 StringBuilder 相比 StringBuffer 可获得 10%~15% 左右的性能提升。

**点评：**

一般候选人的答案是可变性与线程安全性的差别，但其实性能也是很重要的一个点。回答到这个点上，说明你对性能比较敏感，有承担大型网站架构的高可用、高性能方面的潜力。

- 如果要操作少量的数据用 String；
- 单线程操作字符串缓冲区下操作大量数据 StringBuilder；
- 多线程操作字符串缓冲区下操作大量数据 StringBuffer；

**问题 3："int 和 Integer 的区别？"**

**参考答案：**

int 是 java 内置的 8 种基本数据类型之一，而 Integer 是 Java 为 int 对应引入的对应的包装类型（wrapper class）。从 Java 5 开始引入了自动装箱 / 拆箱机制，使得二者可以相互转换。

**点评：**

Java 是一个近乎纯洁的面向对象编程语言，但是为了编程的方便还是引入了基本数据类型，而为了能够将这些基本数据类型当成对象操作，Java 为每一个基本数据类型都引入了对应的包装类型（wrapper class），int 的包装类就是 Integer，从 Java 5 开始引入了自动装箱 / 拆箱机制，使得二者可以相互转换。
Java 为每个原始类型提供了包装类型：

| 原始类型 | boolean | char      | byte | short | int     | long | float | double |
| :------- | :------ | :-------- | :--- | :---- | :------ | :--- | :---- | :----- |
| 包装类型 | Boolean | Character | Byte | Short | Integer | Long | Float | Double |

> 题目变种：
>
> 这个题目的变种通常考察 Integer 的源码实现及自动装箱 / 拆箱机制
>
> 变种 1：给出各 == 运算符的逻辑结果值
>
> ```java
> public static void main(String[] args) {
>  Integer a = new Integer(3);
>  Integer d = new Integer(3);   // 通过new来创建的两个Integer对象
>  Integer b = 3;                  // 将3自动装箱成Integer类型int c = 3;
>  int     c = 3;                  // 基本数据类型3
>  System.out.println(a == b);     // false 两个引用没有引用同一对象
>  System.out.println(a == d);     // false 两个通过new创建的Integer对象也不是同一个引用
>  System.out.println(c == b);     // true b自动拆箱成int类型再和c比较
> }
> ```
>
>  当两边都是 Integer 对象时，是**引用比较**；当其中一个是 int 基本数据类型时，另一个 Integer 对象也会自动拆箱变成 int 类型再进行**值比较**。
>
> 变种 2：给出各 == 运算符的逻辑结果值
>
> ```java
> public static void main(String[] args) {
>  Integer f1 = 100;
>  Integer f2 = 100;
>  Integer f3 = 150;
>  Integer f4 = 150;
> 	System.out.println(f1 == f2);   // true，当int在[-128,127]内时，结果会缓存起来
> 		System.out.println(f3 == f4);   // false，属于两个对象
> }
> ```
>
> 这时很容易认为两个输出要么都是 true 要么都是 false。首先需要注意的是 f1、f2、f3、f4 四个变量都是 Integer 对象引用，所以下面的 == 运算比较的不是值而是引用。装箱的本质是什么呢？当我们给一个 Integer 对象赋一个 int 值的时候，会调用 Integer 类的静态方法 valueOf，关键代码如下：
>
> ```java
> public static Integer valueOf(int i) {
> if (i >= IntegerCache.low && i <= IntegerCache.high)
>  return IntegerCache.cache[i + (-IntegerCache.low)];
> returnnew Integer(i);
> }
> IntegerCache 是 Integer 的内部类。简单的说，如果整型字面量的值在 - 128 到 127 之间，那么不会 new 新的 Integer 对象，而是直接引用常量池中的 Integer 对象，所以上面的面试题中 f1f2 的结果是 true，而 f3f4 的结果是 false。
> ```

**问题 4：“两个对象值相同 (x.equals (y) == true)，但却可有不同的 hash code，这样说对不对？"**

**参考答案：**

不对。如果两个对象 x 和 y 满足 x.equals (y) == true，它们的 hash code 应当相同。

Java 对于 eqauls 方法和 hashCode 方法是这样规定的：(1) 如果两个对象相同（equals 方法返回 true），那么它们的 hashCode 值一定要相同；(2) 如果两个对象的 hashCode 相同，它们并不一定相同。

当然，你未必要按照要求去做，但是如果你违背了上述原则就会发现在使用容器时，相同的对象可以出现在 Set/Map 集合中，同时增加新元素的效率会大大下降（对于使用哈希存储的系统，如果哈希码频繁的冲突将会造成存取性能急剧下降）。

**点评：**

这个题目有很多种问法，比如说 **equals、== 和 hashCode 的区别是什么**等等。但这种问法比较干，不如本题中引入场景来进行发问，当然考察的核心其实一样。

关于 equals 和 hashCode 方法，很多人也就是仅仅知道而已，在 Joshua Bloch 的大作《Effective Java》中是这样介绍 equals 方法的，需要满足：

- 自反性（x.equals (x) 必须返回 true）；
- 对称性（x.equals (y) 返回 true 时，y.equals (x) 也必须返回 true）；
- 传递性（x.equals (y) 和 y.equals (z) 都返回 true 时，x.equals (z) 也必须返回 true）；
- 一致性（当 x 和 y 引用的对象信息没有被修改时，多次调用 x.equals (y) 应该得到同样的返回值），而且对于任何非 null 值的引用 x，x.equals (null) 必须返回 false。

**问题 5：“如果你的 Serializable 类中包含一个不可序列化的成员，会发生什么？如何解决呢？"**

**参考答案**：

任何序列化该类的尝试都会因 NotSerializableException 而失败，但这可以通过在 Java 中给属性设置瞬态 (transient) 变量来轻松解决。

**点评：**

序列化一般和输入输出有关，如读取本地文件，读取远程数据等，典型的如 RPC 调用等。主要考察广度。



## 3. 总结

JDK 基础部分重点是熟练掌握以下几个包的源码：java.lang.* ，java.util.* , [java.io](http://java.io/).* ,java.nio.*，我们后续很多子主题都会对其中的内容作专题讲解，比如说集合部分、并发部分等。正所谓万变不离其宗，对这部分面试题的应对策略就是：

1. 多看源码，多多理解；
2. 职场新人，广度优先，兼顾深度；有经验的候选人，深度优先，兼顾广度。



## 4. 扩展阅读及思考题

### 4.1 扩展阅读

- [一篇文章教会你，如何做到招聘要求中的 “要有扎实的 Java 基础”](https://blog.csdn.net/zuoxiaolong8810/article/details/65629297)
- [java jdk 基础包说明](https://blog.csdn.net/ZYC88888/article/details/90246526)
- [Java 核心技术卷 I 基础知识 + 卷 II 高级特性（第 10 版）](https://item.jd.com/22690417364.html)
- [扩展面试题目_JDK 基础面试题](https://github.com/jiehao2019/imooc_java_interview_questions/blob/master/Java基础技术/JDK基础面试题.md)

### 4.2 思考题

String 类型为什么设计成 final 且不可变的？

- 因为java的String类使用了String pool，字符串常量被多处所引用时，一处进行了修改会影响其它处。
- String被用作HashMap的key是常见的，如果String是可变的，可能会在String被修改后产生两个不同的hashcode，导致丢失对应的value。

关于String pool请看：[What is the concept of String Pool in java? - edureka!](https://www.edureka.co/blog/java-string-pool/)



## 引用/参考

[高薪之路--Java面试题精选集 - jiehao - 慕课网](https://www.imooc.com/read/67)

[Why String is Immutable or Final in Java - Javarevisited](https://javarevisited.blogspot.com/2010/10/why-string-is-immutable-or-final-in-java.html)