# Map（映射）



## 基本信息

### Map类图

![v2-5f93b764594e638e34996430aa76bb11_1440w](https://image-hosting.jellyfishmix.com/20200629202445.jpg)

### 分类

![java-coll](https://image-hosting.jellyfishmix.com/20200629204353.png)

所有的集合框架都包含如下内容：

- 接口：是代表集合的抽象数据类型。例如 Collection、List、Set、Map 等。之所以定义多个接口，是为了以不同的方式操作集合对象
- 实现（类）：是集合接口的具体实现。从本质上讲，它们是可重复使用的数据结构，例如：ArrayList、LinkedList、HashSet、HashMap。
- 算法：是实现集合接口的对象里的方法执行的一些有用的计算，例如：搜索和排序。这些算法被称为多态，那是因为相同的方法可以在相似的接口上有着不同的实现。

### 属性

| Map类             | key        | value      | super       | 说明         |
| ----------------- | ---------- | ---------- | ----------- | ------------ |
| HashMap           | 可以为null | 可以为null | AbstractMap | 线程不安全   |
| HashTable         | 不能为null | 不能为null | Dictionary  | 线程安全     |
| ConcurrentHashMap | 不能为null | 不能为null | AbstractMap | 线程局部安全 |
| TreeMap           | 不能为null | 可以为null | AbstractMap | 线程不安全   |

### 例题

以下说法中正确的有？

- A `StringBuilder` 是线程不安全的。
- B Java类可以同时用 `abstract` 和 `final` 声明。
- C `HashMap` 中，使用 `get(key)==null` 可以判断这个 `Hasmap` 是否包含此key。
- D `volatile` 关键字不保证对变量操作的原子性。

#### 答案

A, D

#### 解析

- A 正确。
- B 错误。`abstract` 需要有实现类来重写此方法，`final` 修饰的方法不可被子类重写。
- C 由于HashMap的value允许null值，因此C错误。有可能存在此key，只不过此key对应的value是null。
- D `volatile` 关键字保证可见性，不保证原子性。



## HashMap

HashMap的底层是数组 + 链表（链表用于解决Hash冲突）。当HashMap的数组长度到达一个临界值的时候（默认初始值为16），就会触发扩容，把所有元素rehash之后再放在扩容后的容器中，这是一个相当耗时的操作。

### 负载因子

- 如果空间利用率高，那么经过的哈希算法计算存储位置的时候，会发现很多存储位置已经有数据了（哈希冲突）。
- 如果为了减小发生哈希冲突的几率，增大数组容量，就会导致空间利用率不高。浪费存储空间。

而负载因子就是表示Hash表中元素的填满程度。

```
负载因子 = 填入表中的元素个数 / 散列表的长度
```

### HashMap初始化大小计算公式
`(存储元素个数/负载因子)+1`

公式“(存储元素个数/负载因子)+1”说明：因为 HashMap 的实际存储量等于：元素个数*负载因子，为了防止 HashMap 扩容，所以公式必须是“(存储元素个数/负载因子)+1”才能防止动态扩容。

#### i.e.

我们已知需要插入 1024 个数据，根据默认的`负载因子 0.75`和公式 `(存储元素个数/负载因子)+1` 得出需要设置的大小为 1367（取整）。

### 阿里巴巴代码规范中对HashMap的规定

> 【推荐】集合初始化时，指定集合初始值大小。
>
> 说明：HashMap 使用 HashMap(int initialCapacity) 初始化，如果暂时无法确定集合大小，那么指定默认值（16）即可。
>
> 正例：initialCapacity = (需要存储的元素个数 / 负载因子) + 1。注意负载因子（即 loader factor）默认为 0.75，如果暂时无法确定初始值大小，请设置为 16（即默认值）。
>
> 反例：HashMap 需要放置 1024 个元素，由于没有设置容量初始大小，随着元素不断增加，容量 7 次被迫扩大，resize 需要重建 hash 表。当放置的集合元素个数达千万级别时，不断扩容会严重影响性能。



## Hashtable，Properties

### Hashtable

Hashtable实现了Map接口，它和HashMap类很相似，但是它支持同步（线程安全）。

像HashMap一样，Hashtable在哈希表中存储键值对 `key - value` 。

像HashMap一样，Hashtable在哈希表中存储键/值对。当使用一个哈希表，要指定用作键的对象，以及要链接到该键的值。

然后，该键经过哈希处理，所得到的散列码被用作存储在该表中值的索引。

### Properties

Properties 继承于 Hashtable，因此支持同步（线程安全）。

表示一个持久的属性集.属性列表中每个键及其对应值都是一个字符串。

被设计用于在一些文件中存储配置和资源。



## 引用

[Java 集合框架 - 菜鸟教程](https://www.runoob.com/java/java-collections.html)

[为什么 HashMap 的加载因子是0.75？ - 蛙课网 - 知乎](https://zhuanlan.zhihu.com/p/149687607)

[阿里巴巴为什么让初始化集合时必须指定大小 - Java中文社群 - 掘金](https://juejin.im/post/5ed06d38f265da770678d0b3)