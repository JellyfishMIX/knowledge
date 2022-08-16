# Lombok



## @Builder

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ResultVO<T> {
    private Integer code;
    private String message;
    private T data;
}
```



## @ToString

包含父类字段

```java
@ToString(callSuper = true)
```



## Q&A

### 1. compile error: cannot be applied to given types;

今天使用lombok的 `@Builder` 时，编译出现以下错误：

```
> Task :compileJava
../demo/src/main/java/com/abc/demo/dto/MyClass.java:12: error: constructor MyClass in class MyClass cannot be applied to given types;
@Builder
^
  required: no arguments
  found: BrowserType,Integer,Integer
  reason: actual and formal argument lists differ in length
Note: ...

1 error

> Task :compileJava FAILED
```

造成错误的 Lombok 设定如下。

```java
@Data
@Builder
@NoArgsConstructor
public class MyClass {
    ...
}
```

解決方式一是移除 `@NoArgsConstructor`。

```java
@Data
@Builder
public class MyClass {
    ...
}
```

解決方式二是加上 `@AllArgsConstructor`。

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MyClass {
    ...
}
```

错误原因可能是因为 `＠Builder` 自带的全参构造被 `@NoArgsConstructor` 覆写所導致。



## 引用/参考

[Lombok @Builder 编译错误 compile error - 菜鸟工程师 - 肉猪的博客](https://matthung0807.blogspot.com/2019/11/lombok-builder-compile-error.html)