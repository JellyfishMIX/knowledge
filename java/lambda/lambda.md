# lambda



## method

分为终结符和非终结符。非终结符会返回一个 pipeline。pipeline 接收上一个 pipeline 传递过来的数据。

虽然使用 lambda 会导致性能下降，比如 for 循环中边遍历边过滤效率会更高，但以效率为损失换来的可读性是目前普遍的做法。

在写一些注重性能的算法和底层库的时候不会使用这种损失性能提高可读性的方法。

```mermaid
graph LR
A[操作] --> B[状态]
B --> C["有状态(stateful)"]
B --> D["无状态(stateless)"]
C --> E[sorted]
C --> F[skip]
C --> G[limit]
C --> H[任何触发状态变化的程序]
D --> I[map]
D --> J[reduce]
A --> K[副作用]
K --> L["纯函数(pure function)"]
K --> M[非纯函数]
```

### map

映射，非终结符。

### filter

过滤，非终结符。

### forEach

遍历，终结符。



## 函数式编程基本概念

```mermaid
graph LR
A[函数式编程基本概念] --> B[类型]
B --> C["int -> String"]
C --> C1[toString]
B --> D["(int) -> int[]"]
D --> D1["int[]::new (把一些int 转换为 int[])"]
B --> E["高阶函数: int -> int -> int"]
E --> E1["Integer::max (输入一个int，返回一个 int -> int 的函数，高阶函数返回的还是函数)"]
B --> F[类型类]
A --> G[性质]
G --> H[immutable/pure]
H --> H1["在写 pure function 的时候不可避免地需要把数据结构设置为 immutable"]
H1 --> H11["immutable 方便调试"]
H1 --> H12["数据不会发生变化，像常量一样，不容易出 bug"]
H1 --> H13["有很好的并发能力"]
H1 --> H14["大数据领域有很多 immutable 的应用"]
G --> I[lazy]
I --> I1["触发终结操作时才会进行计算，非终结操作并不会进行计算"]
G --> J[safy]
J --> J1["函数式编程相对安全，因为有数学理论基础做保障"]
J --> J2["monad 架构"]
```



## @FunctionaInterface

把 lambda expression 转换为 interface。



## Variable used in lambda expression should be final or effectively final

### 问题如图所示

![image-20211004181312460](https://image-hosting.jellyfishmix.com/20211004181312.png)

修改后：

![image-20211004181320709](https://image-hosting.jellyfishmix.com/20211004181320.png)

### 问题简化表述

```java
for (int index; index < 1000; index ++) {
    threadPool.execute(() -> {
    	System.out.println(index);
    });
}
```

报错，需要修改成：

```java
for (int index; index < 1000; index ++) {
	int finalIndex = index;
    threadPool.execute(() -> {
    	System.out.println(finalIndex);
    });
}
```

报错内容：

```
Variable used in lambda expression should be final or effectively final
```

从字面上来理解这句话，意思是：*lambda表达式中使用的变量应该是final或者有效的final*，也就是说，lambda 表达式只能引用标记了 final 的外层局部变量，这就是说不能在 lambda 内部修改定义在域外的局部变量，否则会编译错误。

原因：在lambda表达式中对变量的操作都是基于原变量的副本，不会影响到原变量的值。

假定没有要求lambda表达式外部变量为final修饰，那么开发者会误以为外部变量的值能够在lambda表达式中被改变，而这实际是不可能的，所以要求外部变量为final是在编译期以强制手段确保用户不会在lambda表达式中做修改原变量值的操作。

为什么会有这种规定？

其实在 Java 8 之前，匿名类中如果要访问局部变量的话，那个局部变量必须显式的声明为 final。

详细请看：[【Java异常】Variable used in lambda expression should be final or effectively final - No8g攻城狮 - CSDN](https://blog.csdn.net/weixin_44299027/article/details/117333667)

### 引用/参考

[【Java异常】Variable used in lambda expression should be final or effectively final - No8g攻城狮 - CSDN](https://blog.csdn.net/weixin_44299027/article/details/117333667)