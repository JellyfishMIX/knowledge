# Lambda表达式



## 方法引用

诸如 `String::length` 的语法形式叫做方法引用（*method references*），这种语法用来替代某些特定形式Lambda表达式。如果Lambda表达式的全部内容就是调用一个已有的方法，那么可以用方法引用来替代Lambda表达式。方法引用可以细分为四类：

| 方法引用类别       | 举例             |
| :----------------- | :--------------- |
| 引用静态方法       | `Integer::sum`   |
| 引用某个对象的方法 | `list::add`      |
| 引用某个类的方法   | `String::length` |
| 引用构造方法       | `HashMap::new`   |



##  lambda求交集、差集、并集、去重并集

```java
import java.util.ArrayList;
import java.util.List;
import static java.util.stream.Collectors.toList;
 
public class Test {
 
    public static void main(String[] args) {
        List<String> list1 = new ArrayList<String>();
        list1.add("1");
		list1.add("2");
		list1.add("3");
		list1.add("5");
		list1.add("6");
 
        List<String> list2 = new ArrayList<String>();
        list2.add("2");
		list2.add("3");
		list2.add("7");
		list2.add("8");
 
        // 交集
        List<String> intersection = list1.stream().filter(item -> list2.contains(item)).collect(toList());
        System.out.println("---交集 intersection---");
        intersection.parallelStream().forEach(System.out :: println);
 
        // 差集 (list1 - list2)
        List<String> reduce1 = list1.stream().filter(item -> !list2.contains(item)).collect(toList());
        System.out.println("---差集 reduce1 (list1 - list2)---");
        reduce1.parallelStream().forEach(System.out :: println);
 
        // 差集 (list2 - list1)
        List<String> reduce2 = list2.stream().filter(item -> !list1.contains(item)).collect(toList());
        System.out.println("---差集 reduce2 (list2 - list1)---");
        reduce2.parallelStream().forEach(System.out :: println);
 
        // 并集
        List<String> listAll = list1.parallelStream().collect(toList());
        List<String> listAll2 = list2.parallelStream().collect(toList());
        listAll.addAll(listAll2);
        System.out.println("---并集 listAll---");
        listAll.parallelStream().forEachOrdered(System.out :: println);
 
        // 去重并集
        List<String> listAllDistinct = listAll.stream().distinct().collect(toList());
        System.out.println("---得到去重并集 listAllDistinct---");
        listAllDistinct.parallelStream().forEachOrdered(System.out :: println);
 
        System.out.println("---原来的List1---");
        list1.parallelStream().forEachOrdered(System.out :: println);
        System.out.println("---原来的List2---");
        list2.parallelStream().forEachOrdered(System.out :: println);
 
    }
}
```



## 使用lambda表达式进行Integer[]转int[]

```java
int[] intArr = list.stream().mapToInt(Integer::intValue).toArray();
```



## filter()

```java
    @Test
    public void testFilter() {
        List<User> list = new ArrayList<>();
        User user1 = new User("张三", 15, "男");
        User user2 = new User("李四", 22, "男");
        User user3 = new User("王五", 25, "男");
        User user4 = new User("赵六", 30, "男");
        User user5 = new User("李世民", 28, "男");
        User user6 = new User("杨贵妃", 18, "女");
        User user7 = new User("貂蝉", 16, "女");
        list.add(user1);
        list.add(user2);
        list.add(user3);
        list.add(user4);
        list.add(user5);
        list.add(user6);
        list.add(user7);

        List<User> collect = list.stream()
                .filter(user -> !"张三".equals(user.getName()))
                .collect(Collectors.toList());
        // 张三不被输出，非张三被输出
        System.out.println(collect);
    }
```

满足 filter() 中条件的数据会被留下来。



## 引用/参考

[关于Java Lambda表达式看这一篇就够了](https://objcoding.com/2019/03/04/lambda/)

[java8两个List集合取交集、并集、差集、去重并集 - lizhiyong - 掘金](https://juejin.im/post/6844903833726894093#comment)