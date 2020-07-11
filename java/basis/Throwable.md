# Throwable



## Throwable的分类

![20171027131229769](https://image-hosting.jellyfishmix.com/20200618164122.jpeg)



## Exception

 Exception是异常，他又分为。checked exception和unchecked exception。checked exceptions必须显式处理，运行时一场由JVM处理，

- 可查的异常（checked exception）:Exception下除了RuntimeException外的异常。
- 不可查的异常（unchecked exception）:RuntimeException及其子类。

![img](https://uploadfiles.nowcoder.com/images/20180704/3807435_1530666258064_20577AE82E2EC5D6D44DD2CA01C99BBA)

### 异常处理

- 运行时异常可以不处理。当出现这样的异常时，总是由虚拟机接管。比如我们从来没有人去处理过NullPointerException，它就是运行时异常。
- 出现运行时异常后，系统会把异常一直往上层抛，一直遇到处理代码。如果没有处理块，到最上层。
  - 如果是多线程就由Thread.run()抛出，线程退出。
  - 如果是单线程就被main()抛出，主程序退出。
- RuntimeException是Exception的子类，也有一般异常的特点，是可以被try catch处理的，只不过通常不对它处理。即，如果不对RuntimeException进行处理，那么出现运行时异常之后，要么是线程中止，要么是主程序终止。

#### try catch和finally

```java
try {
  
} catch () {
  
} finally {
  
}
```

- finally这个名字的真正含义是指，从try代码块出来才一定会执行相应的finally代码块。

- 即使未发生异常，只要从try catch块中出来，finally就会执行。

- finally一定会在return之前执行，但是如果finally使用了return或者throw语句，将会使try和catch中的return或者throw失效。

- finally先会把try或者catch代码块中的返回值保留，再来执行finally代码块中的语句，等到finally代码块执行完毕之后，在把之前保留的返回值给返回出去。

- 碰到finally的时候，编译器做的事情其实不仅仅是调整代码顺序，而是复制finally块的代码。这一块代码会被复制到每个try块中的出口之前，包括return, throw exception，甚至是外层for的break。

  而这里的出口不是指一条java语句，而是编译过之后的jump指令，所以如果return f(x)，编译过之后会变成：

  ```
  f(x)的汇编码
  finally的汇编码
  jump 上层调用地址
  ```

#### 例题

##### 1.

如下代码的输出是：

```java
package Test;
public class Test {
    private static void test(int[] arr) {
        for (int i = 0; i < arr.length; i++) {
            try {
                if (arr[i] % 2 == 0) {
                    throw new NullPointerException();
                } else {
                    System.out.print(i);
                }
            } finally {
                System.out.print("e");
            }
        }
    }
 
    public static void main(String[]args) {
        try {
            test(new int[] {0, 1, 2, 3, 4, 5});
        } catch (Exception e) {
            System.out.print("E");
        }
    }
}
```

- A 编译出错
- B eE
- C Ee
- D eE1eE3eE5
- E Ee1Ee3Ee5

###### 答案

B

###### 解析

- 由于arr[0] =0，所以在进入 test()方法里面会在第一个if处抛出一个 NullPointerException，由于本方法内此异常没有被catch，会向上给调用处抛出。
- 接着会执行 finally的语句， (finally语句先于try中的return 执行)，输出一个"e"，然后回到调用处main方法中，捕捉到异常，所以进入到catch语句中，打印一个"E",所以最终结果为"eE"。



## 参考

[@Transactional(rollbackFor=Exception.class)的使用](https://blog.csdn.net/Mint6/article/details/78363761)

[JAVA中finally之前有return语句该如何执行？ - Charlie W的回答 - 知乎](https://www.zhihu.com/question/62447192/answer/198532283)