# [转]Java Scanner next()和nextLine()的区别



## Scanner简单介绍：

Scanner的用途广泛，而且好用，它自身包含了很多构造方法，可以接收各种类型数据，可以是一个文件、输入流、控制台…… 
Scanner为我们提供了很多的方法以使用，其中有两个方法next()和nextLine()，这两个方法返回的都是String类型的，那么为什么要有两个功能类似的方法，它们的区别又在哪里呢？



## next()特点：

1. 一定要读取到有效字符后才可以结束输入。

2. 对输入有效字符之前遇到的空白，next()方法会自动将其去掉。
3. 只有输入有效字符后才将其后面输入的空白作为分隔符或者结束符。 
   next()不能得到带有空格的字符串 。



## nextLine()特点：

1. 以Enter为结束符,也就是说nextLine()方法返回的是输入回车之前的所有字符。
2. 可以获得空白。



## 举例

例如：

```java
public static void main(String[] args) {
    Scanner sc = new Scanner(System.in);        
    //next方式接收字符串
    System.out.println("next方式接收：");
    String nextStr = sc.next();
    String nextStr2 = sc.next();
    System.out.println("next()输入结果:\n"+nextStr+nextStr2);
    //nextLine方式接收字符串
    System.out.println("nextLine方式接收：");
    String nextLineStr = sc.nextLine();
    System.out.println("第二个nextLine：");
    String nextLineStr2 = sc.nextLine();
    System.out.println("nextLine()输入结果:"+nextLineStr+"\n"+nextLineStr2);
    sc.close();
}
```
运行结果： 
next方式接收： 
`i’m haydn`
next()输入结果: 
`i’mhaydn`
nextLine方式接收： 
第二个nextLine： 
`i’m haydn`
nextLine()输入结果: 
`i’m haydn`

总结：虽然这里只列出了控制台输入的这一种方法，但是其他方式获得的结果和结论是一致的。掌握两者之间的区别，以便更好的利用这两个方法。



## 转自

版权声明：本文为CSDN博主「孙海峰VIP」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/shf4715/article/details/46810475

