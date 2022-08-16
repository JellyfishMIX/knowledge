# foreach 和 for



## 区别

1.在长度固定或长度不需要计算的时候for循环效率高于foreach,长度不确定时使用foreach。

2.foreach只用于遍历数组和集合元素（工作原理：逐个枚举出数组或集合中的每一个元素）。

3.foreach只读，不能修改元素的值，for可以给元素赋值。

4.foreach通过内部指针自增遍历数组。