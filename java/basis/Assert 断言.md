# Assert 断言



## 语法

```java
assert condition;
```

condition是一个必须为真(true)的表达式。如果表达式的结果为true，那么断言为真，并且无任何行动
如果表达式为false，则断言失败，则会抛出一个AssertionError对象。

```java
assert condition : string;
```

冒号后跟的是一个字符串，通常用于断言失败后的提示信息。它是一个传到AssertionError构造函数的值，如果断言失败，该值被转化为它对应的字符串，并显示出来。



##  注意事项

- Java在执行程序的时候默认是不启动断言检查的，即所有的断言语句都将被忽略。
- 如果要开启断言检查，则需要使用 `-enableassertions` 或 `-ea` JVM参数(`VM options`)来开启；如果要手动忽略断言检查，则可以通过使用 `-disableassertions` 或 `-da` JVM参数来忽略断言语句。

