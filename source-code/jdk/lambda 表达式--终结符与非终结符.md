## stream 流式计算 api--终结符与非终结符



## stream 流式计算 api 

分为终结符和非终结符。非终结符会返回一个 pipeline。pipeline 接收上一个 pipeline 传递过来的数据。

### 优点

1. 可读性高：stream 流式计算每个 api 的动作特点是很鲜明的，见名知意。比如 map 会把输入流的元素数据类型转换成另一种数据类型，然后产生新的输出流。peek 是把流中元素拿出来消费一下，并不改变输入流中的元素等。不过这种可读性高需要熟悉 stream 的 api 后才能感受到。
2. 函数式风格：对流的操作会产生一个结果，但流的数据源不会被修改。
3. 延迟计算：stream 流 api 分为终结符与非终结符，非终结符中传入的函数，并不会立刻执行。在遇到终结符时才会触发计算。
4. 代码简洁：stream 流把一些通用的动作封装进了 api，调用 api 能节省一些代码。

### 缺点

1. 和传统的 iterator 迭代相比，stream 流式计算会导致性能下降，比如每次输入流与输出流都要做一次拆开和组装的动作，这是有额外性能开销的。
2. 相比于好处，目前普遍认为性能的降低是可以接受的，更何况对性能的降低并不多。
3. 在写一些框架级别注重性能的算法，和底层数据库的时候，一般不会使用这种损失性能提高可读性的方法。底层注重性能的代码，开发者一般偏好追求极致的性能。



## 终结符与非终结符

stream 流 api 分为终结符与非终结符。非终结符接收一个 stream，并允许传入一个函数 fn 对流中的元素进行处理，返回一个 stream，传递给下一个 api。非终结符中传入的函数，并不会立刻执行，在遇到终结符时才会触发计算。

非终结符例如：

1. map 映射
2. filter 过滤

终结符例如：

1. forEach 遍历
2. toArray() 把流中元素收集转成数组。
3. collect(Collectors.toList()) 把流中元素收集转成 List

### demoA，没有触发终结符

```java
public class TerminatorTest {
    @Test
    public void terminator() {
        String[] words = new String[]{"Hello","World"};
        Counter counter = new Counter();
        Arrays.stream(words)
                .map((outerElement) -> {
                    try {
                        return outerElement.chars().peek((innerElement) -> {
                            // 如果内层的这个函数执行了，会触发 counter 计数
                            counter.selfIncrease();
                        });
                    } finally {
                        System.out.println("finally 中感知的 count:" + counter.getCount());
                    }
                })
                .peek(intStream -> {
                    // System.out.println(Arrays.toString(intStream.toArray()));
                })
                .collect(Collectors.toList());
    }
}
```

外层一个 stream 流遍历，内层还有一个新的 stream 流遍历，内层 stream 遍历过程中会用到一个函数，如果内层的这个函数执行了，会触发 counter 计数。

在这段代码中，内层流没有触发终结符，就返回给外层流遍历过程。外层流也没有触发内层流的终结符，导致内层流函数没有被触发。

执行结果

```
finally 中感知的 count:0
finally 中感知的 count:0
```

如果内层的这个函数执行了，会触发 counter 计数。执行结果符合猜想，没有触发 counter 计数，证明内层函数没有被触发。

### demoB，触发了终结符

```java
    @Test
    public void terminatorA() {
        String[] words = new String[]{"Hello","World"};
        Counter counter = new Counter();
        Arrays.stream(words)
                .map((outerElement) -> {
                    try {
                        // 内层这个函数会延迟计算
                        return outerElement.chars().peek((innerElement) -> {
                            // 如果内层的这个函数执行了，会触发 counter 计数
                            counter.selfIncrease();
                        });
                    } finally {
                        // finally 想感知内层的这个函数执行过程中的信息
                        System.out.println("finally 中感知的 count:" + counter.getCount());
                    }
                })
                .peek(intStream -> {
                    System.out.println(Arrays.toString(intStream.toArray()));
                })
                .collect(Collectors.toList());
    }
```

我们修改一下代码，在内层流返回给外层流遍历过程后，触发一下内层流的终结符。可以看到在内层流的终结符被触发后，内层流使用的函数被执行了。

执行结果

```
finally 中感知的 count:0
[72, 101, 108, 108, 111]
finally 中感知的 count:5
[87, 111, 114, 108, 100]
```

### try finally 包裹一个延迟计算的函数的问题

发现了一个问题，try finally 包裹一个延迟计算的函数，在 finally 中不能感知这个函数执行过程中的信息。因为代码返回进入到 finally 时，返回的是一个流。流中的函数没有被执行，finally 无从谈起去看函数执行过程中的信息。

#### 解决方案

```java
    @Test
    public void terminatorB() {
        String[] words = new String[]{"Hello","World"};
        Counter counter = new Counter();
        Arrays.stream(words)
                .map((outerElement) -> {
                    try {
                        // 这个函数会立刻执行
                        outerElement.chars().forEach((innerElement) -> {
                            // 如果内层的这个函数执行了，会触发 counter 计数
                            counter.selfIncrease();
                        });
                        return outerElement;
                    } finally {
                        // finally 想感知内层的这个函数执行过程中的信息
                        System.out.println("finally 中感知的 count:" + counter.getCount());
                    }
                })
                .collect(Collectors.toList());
    }
```

执行结果

```
finally 中感知的 count:5
finally 中感知的 count:10
```

内层的函数从延迟计算改为立刻执行，try finally 中就可以感知到这个函数执行过程中的信息了。