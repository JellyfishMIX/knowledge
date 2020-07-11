# enum（枚举类）



## 引言

枚举类型是 JDK 5 之后引进的一种非常重要的引用类型，可以用来定义一系列枚举常量。

在没有引入 enum 关键字之前，要表示可枚举的变量，只能使用 `public static final` 的方式。

```java
public staic final int SPRING = 1;
public staic final int SUMMER = 2;
public staic final int AUTUMN = 3;
public staic final int WINTER = 4;
```

这种实现方式有几个弊端。首先，类型不安全。试想一下，有一个方法期待接受一个季节作为参数，那么只能将参数类型声明为 int，但是传入的值可能是 99。显然只能在运行时进行参数合理性的判断，无法在编译期间完成检查。其次，指意性不强，含义不明确。我们使用枚举，很多场合会用到该枚举的字串符表达，而上述的实现中只能得到一个数字，不能直观地表达该枚举常量的含义。当然也可用 String 常量，但是又会带来性能问题，因为比较要依赖字符串的比较操作。

使用 enum 来表示枚举可以更好地保证程序的类型安全和可读性。

enum 是类型安全的。除了预先定义的枚举常量，不能将其它的值赋给枚举变量。这和用 int 或 String 实现的枚举很不一样。

enum 有自己的名称空间，且可读性强。在创建 enum 时，编译器会自动添加一些有用的特性。每个 enum 实例都有一个名字 (name) 和一个序号 (ordinal)，可以通过 toString() 方法获取 enum 实例的字符串表示。还以通过 values() 方法获得一个由 enum 常量按顺序构成的数组。

enum 还有一个特别实用的特性，可以在 switch 语句中使用，这也是 enum 最常用的使用方式了。

下面我们从源码方面分析一下 enum 的实现方式，并介绍几种 enum 的用法。



## 反编译枚举类型源码

```java
public enum Season {
  SPRING, SUMMER, AUTUMN, WINTER;
}
```

用 javap 反编译一下生成的 class 文件：

```java
public final class Season extends java.lang.Enum<Season> {
  public static final Season SPRING;
  public static final Season SUMMER;
  public static final Season AUTUMN;
  public static final Season WINTER;
  public static Season[] values();
  public static Season valueOf(java.lang.String);
  static {};
}
```

可以看到，实际上在经过编译器编译后生成了一个 Season 类，该类继承自 Enum 类，且是 final 的。从这一点来看，Java 中的枚举类型似乎就是一个语法糖。

每一个枚举常量都对应类中的一个 `public static final` 的实例，这些实例的初始化应该是在 static {} 语句块中进行的。因为枚举常量都是 final 的，因而一旦创建之后就不能进行更改了。 此外，Season 类还实现了 `values()` 和 `valueOf()` 这两个静态方法。

再用 jad 进行反编译，我们可以大致看到 Season 类内部的实现细节：

```java
public final class Season extends Enum
{

    public static Season[] values()
    {
        return (Season[])$VALUES.clone();
    }

    public static Season valueOf(String s)
    {
        return (Season)Enum.valueOf(Season, s);
    }

    private Season(String s, int i)
    {
        super(s, i);
    }

    public static final Season SPRING;
    public static final Season SUMMER;
    public static final Season AUTUMN;
    public static final Season WINTER;
    private static final Season $VALUES[];

    static 
    {
        SPRING = new Season("SPRING", 0);
        SUMMER = new Season("SUMMER", 1);
        AUTUMN = new Season("AUTUMN", 2);
        WINTER = new Season("WINTER", 3);
        $VALUES = (new Season[] {
            SPRING, SUMMER, AUTUMN, WINTER
        });
    }
}
```

除了对应的四个枚举常量外，还有一个私有的数组，数组中的元素就是枚举常量。编译器自动生成了一个 private 的构造方法，这个构造方法中直接调用父类的构造方法，传入了一个字符串和一个整型变量。从初始化语句中可以看到，字符串的值就是声明枚举常量时使用的名称，而整型变量分别是它们的顺序（从0开始）。枚举类的实现使用了一种多例模式，只有有限的对象可以创建，无法显示调用构造方法创建对象。

`values()` 方法返回枚举常量数组的一个浅拷贝，可以通过这个数组访问所有的枚举常量；而 `valueOf()`则直接调用父类的静态方法 `Enum.valueOf()`，根据传入的名称字符串获得对应的枚举对象。

Enum 类是不能被继承的，如果我们按照上面反编译的结果自己写一个这样的实现，是不能编译成功的。Java 编译器限制了我们显式的继承 `java.Lang.Enum` 类, 报错 `The type may not subclass Enum explicitly`。



## 例题

### 1. 

```java
enum AccountType
{
    SAVING, FIXED, CURRENT;
    private AccountType()
    {
        System.out.println(“It is a account type”);
    }
}
class EnumOne
{
    public static void main(String[]args)
    {
        System.out.println(AccountType.FIXED);
    }
}
```

最后输出：

```java
It is a account type
It is a account type
It is a account type
AccountType
```

#### 解析

`自定义枚举类`在编译后会生成一个`编译后的类`，此`编译后的类`继承自`Enum`类。此`编译后的类`中有一个`static`代码块，会调用n次`编译后的类`的构造方法（n为`自定义枚举类`中实例的个数）。



## 引用

[Java 枚举源码分析](https://blog.jrwang.me/2016/java-enum/#反编译枚举类型源码)