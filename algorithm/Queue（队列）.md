## Queue（队列）



## 优先队列

不同实现的时间复杂度

|              | 入队    | 出队    |
| ------------ | ------- | ------- |
| 普通线性结构 | O(1)    | O(n)    |
| 顺序线性结构 | O(n)    | O(1)    |
| 堆           | O(logn) | O(logn) |

Java的预置优先队列使用最小堆实现。



## 广义队列

拥有以下功能：

```java
public interface Queue<E> {
    int getSize();
    boolean isEmpty();

    /**
     * 入队
     *
     * @param e element
     */
    void enqueue(E e);

    /**
     * 出队。Get the element at the head of the queue and remove it from the queue.
     * @return element at the head of the queue.
     */
    E dequeue();

    /**
     * 获取队首元素。Get the element at the head of the queue but not remove it from the queue.
     * @return element at the head of the queue.
     */
    E getFront();
}
```



