## lambda 表达式--map 与 flatMap 的区别



## 前言

本文将通过案例讲解。



## 目标

对给定单词列表 ["Hello","World"]，你想做字符的去重，返回列表["H","e","l","o","W","r","d"]



## 使用 map 与 flatMap 两种方式

```java
public void flatMapTest() {
    String[] words = new String[]{"Hello","World"};
    List<String[]> a = Arrays.stream(words)
            .map(word -> word.split(""))
            .distinct()
            .collect(Collectors.toList());
    a.forEach(System.out::print);

    System.out.println();
    System.out.println("-----------------");

    List<String> b = Arrays.stream(words)
            .map(word -> word.split(""))
            .flatMap(Arrays::stream)
            .distinct()
            .collect(Collectors.toList());
    b.forEach(System.out::print);
}
```

打印结果

```
[Ljava.lang.String;@dd3b207[Ljava.lang.String;@551bdc27
-----------------
HeloWrd
```

发现使用 map 的方式，流处理结束后得到的是一个 String[]。而使用 flatMap 处理结束后得到的是 List<String>。



## map

map 的方式处理过程如下：

![img](https://image-hosting.jellyfishmix.com/20220928145934.png)

1. Arrays.stream(words) 得到两个流，"Hello"和"World"。此时每个流中元素的数据类型是 String。
2. map(word -> word.split("")) 把之前两个流中的 String 进行切分，切分后每个流中元素的数据类型是 String[]。
3. distinct() 去重，只会以流为单位做去重，即把上一步得到的两个 String[] 对象之间做去重，并不会对 String[] 里的字符串元素做去重。这一步处理完成后，还是两个流，每个流中元素的数据类型是 String[]。
4. collect(Collectors.toList()) 收集后，收集到的是两个 String[] 对象。



## flatMap

flatMap 的方式处理过程如下：

![img](https://image-hosting.jellyfishmix.com/20220928150643.png)

1. Arrays.stream(words) 得到两个流，"Hello"和"World"。此时每个流中元素数据类型是 String。
2. map(word -> word.split("")) 把之前两个流中的 String 进行切分，切分后每个流中元素的数据类型是 String[]。
3. flatMap(Arrays::stream) 把接收到的每一个流，使用 Arrays::stream 这个函数，生产出新的多个流。之前输入流中元素的数据类型是 String[]，经过 Arrays::stream 的处理后，把这个元素转换成了多个元素。这转换后的多个元素，每个元素做成了每个新的流。具体地说，输入流中的元素 String[]，被 Arrays::stream 转换成了多个元素，每个元素是一个 String，然后这多个元素被做成了新的流。这一步处理完成后，流中元素的数据类型是 String。
4. distinct() 去重，只会以流为单位做去重。接收到的流中元素是 String，会针对接收到的所有输入流中的 String 做去重。这一步处理完成后，流数量会减少，被干掉的流是所含元素重复的。这一步处理完成后，流中元素的数据类型是 String。
5. collect(Collectors.toList()) 收集后，收集到的是多个 String。



## Arrays::stream

用到了这个函数 Arrays::stream，这里做一下简单的说明。

```java
public class Arrays {
    public static <T> Stream<T> stream(T[] array) {...}
}
```

此方法把入参数组中的每个元素做成一个流返回。



## 总结

map(mapper)，接收一个流，使用函数 mapper 针对流中元素做转换，产生一个新的流。流数量不变。

flatMap(mapper)，接收一个流，使用函数 mapper 针对流中元素做处理，并生产出一个或多个新的流。流的数量不变或增加。