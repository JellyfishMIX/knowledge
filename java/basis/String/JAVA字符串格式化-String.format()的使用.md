# JAVA字符串格式化-String.format()的使用



String类的format()方法用于创建格式化的字符串以及连接多个字符串对象。熟悉C语言的同学应该记得C语言的sprintf()方法，两者有类似之处。format()方法有两种重载形式。

format(String format, Object... args) 新字符串使用本地语言环境，制定字符串格式和参数生成格式化的新字符串。

format(Locale locale, String format, Object... args) 使用指定的语言环境，制定字符串格式和参数生成格式化的字符串。

显示不同转换符实现不同数据类型到字符串的转换，如图所示。

 

| 转 换 符 | 说  明                                      | 示  例       |
| -------- | ------------------------------------------- | ------------ |
| %s       | 字符串类型                                  | "mingrisoft" |
| %c       | 字符类型                                    | 'm'          |
| %b       | 布尔类型                                    | true         |
| %d       | 整数类型（十进制）                          | 99           |
| %x       | 整数类型（十六进制）                        | FF           |
| %o       | 整数类型（八进制）                          | 77           |
| %f       | 浮点类型                                    | 99.99        |
| %a       | 十六进制浮点类型                            | FF.35AE      |
| %e       | 指数类型                                    | 9.38e+5      |
| %g       | 通用浮点类型（f和e类型中较短的）            |              |
| %h       | 散列码                                      |              |
| %%       | 百分比类型                                  | ％           |
| %n       | 换行符                                      |              |
| %tx      | 日期与时间类型（x代表不同的日期与时间转换符 |              |

测试用例

![复制代码](https://image-hosting.jellyfishmix.com/20220323214553.gif)

```
    public static void main(String[] args) {
        String str=null;
        str=String.format("Hi,%s", "王力");
        System.out.println(str);
        str=String.format("Hi,%s:%s.%s", "王南","王力","王张");          
        System.out.println(str);                         
        System.out.printf("字母a的大写是：%c %n", 'A');
        System.out.printf("3>7的结果是：%b %n", 3>7);
        System.out.printf("100的一半是：%d %n", 100/2);
        System.out.printf("100的16进制数是：%x %n", 100);
        System.out.printf("100的8进制数是：%o %n", 100);
        System.out.printf("50元的书打8.5折扣是：%f 元%n", 50*0.85);
        System.out.printf("上面价格的16进制数是：%a %n", 50*0.85);
        System.out.printf("上面价格的指数表示：%e %n", 50*0.85);
        System.out.printf("上面价格的指数和浮点数结果的长度较短的是：%g %n", 50*0.85);
        System.out.printf("上面的折扣是%d%% %n", 85);
        System.out.printf("字母A的散列码是：%h %n", 'A');
    }
```

![复制代码](https://image-hosting.jellyfishmix.com/20220323214553.gif)

  

## 引用/参考

（转载自：http://www.cnblogs.com/happyday56/p/3996498.html）