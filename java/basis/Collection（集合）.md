# Collection（集合）



##  基本信息

### UML图

![2243690-9cd9c896e0d512ed](https://image-hosting.jellyfishmix.com/20200629203809.jpg)

### 分类

![java-coll](https://image-hosting.jellyfishmix.com/20200629203645.png)

所有的集合框架都包含如下内容：

- **接口：**是代表集合的抽象数据类型。例如 Collection、List、Set、Map 等。之所以定义多个接口，是为了以不同的方式操作集合对象。
- **实现（类）：**是集合接口的具体实现。从本质上讲，它们是可重复使用的数据结构，例如：ArrayList、LinkedList、HashSet、HashMap。
- **算法：**是实现集合接口的对象里的方法执行的一些有用的计算，例如：搜索和排序。这些算法被称为多态，那是因为相同的方法可以在相似的接口上有着不同的实现。



## Set

### 基本信息

保存的数据不重复。



## ArrayList

### 例题

#### 1.

list是一个ArrayList的对象，哪个选项的代码填到//todo delete处，可以在Iterator遍历的过程中正确并安全的删除一个list中保存的对象？

```java
Iterator it = list.iterator();
int index = 0;
while (it.hasNext())
{
    Object obj = it.next();
    if (needDelete(obj))  //needDelete返回boolean，决定是否要删除
    {
        //todo delete
    }
    index ++;
}
```

- A it.remove();
- B list.remove(obj);
- C list.remove(index);
- D list.remove(obj,index);

##### 答案

A

##### 解析

Iterator 支持从源集合中安全地删除对象，只需在 Iterator 上调用 remove() 即可。这样做的好处是可以避免 ConcurrentModifiedException ，当打开 Iterator 迭代集合时，同时又在对集合进行修改。有些集合不允许在迭代时删除或添加元素，但是调用 Iterator 的 remove() 方法是个安全的做法。 

ArrayList 继承了 AbstractList， 其中 AbstractList 中有个 modCount 代表了集合修改的次数。在ArrayList的iterator方法中会判断 expectedModCount 与 modCount 是否相等，如果相等继续执行，不相等报错。只有iterator的 remove() 方法在调用自身的 remove() 之后让 expectedModCount 与modCount 相等，所以是安全的。



## 引用

