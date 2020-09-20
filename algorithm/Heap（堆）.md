# Heap（堆）



## 优先队列

- 普通队列：先进先出，后进后出。

- 优先队列：出队顺序和入队顺序无关，和优先级相关。



## 最大堆

把元素一层一层地排列，直到排列完为止。

堆中某个节点的值总是不大于其父节点的值。



## 最小堆

把元素一层一层地排列，直到排列完为止。

堆中某个节点的值总是不小于其父节点的值。



## 用数组存储二叉堆

### 数组索引从1开始

![截屏2020-07-03下午11.04.49](https://image-hosting.jellyfishmix.com/20200703231203.png)

```
parent = i / 2;（整数除法，余数抹去）

left child (i) = 2 * i;
right child (i) = 2 * i + 1;
```

### 数组索引从0开始

![截屏2020-07-03下午11.15.27](https://image-hosting.jellyfishmix.com/20200703231558.png)

```
parent (i) = (i - 1) / 2;

left child (i) = 2 * i + 1;
right child (i) = 2 * i + 2;
```

### Sift Up

![截屏2020-07-14上午7.36.59](https://image-hosting.jellyfishmix.com/20200714073826.png)

上浮操作，把新添加的元素逐级和和父亲节点比较，子节点大于父亲节点则交换。

### Sift Down

![截屏2020-09-20下午11.44.07](https://image-hosting.jellyfishmix.com/20200920234431.png)

下沉操作。父结点依次和左右孩子比较，与左右孩子中最大的那个进行替换。