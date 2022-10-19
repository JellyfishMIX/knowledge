# HashMap 源码分析



## 说明

本文基于 jdk 8 编写。



## HashMap 的结构

![HashMap 大致结构](https://image-hosting.jellyfishmix.com/20220729155057.png)

1. 图中的数组是 table 属性，hashMap 基础的属性。一个数组，用于承载 node，table 的每一个格被称为桶。

2. node 是 hashMap 中基础的 node 节点，用于存储 key, value。

3. 桶位置计算的公式是 `(n - 1) & hash`，n 指 table 的长度，hash 指 key 的 hash 值。
4. 桶位置计算时有可能出现 hash 冲突的现象，在 jdk 1.7 及之前采用的是把 node 拼接成链表的方式。但如果 hash 冲突严重，桶位置上的链表会很长，影响查询性能。从 jdk 1.8 开始，改成了链表 + 红黑树的方式，在一个桶位置上元素很多的情况下，树的查询效率优于链表。



## 关键属性

### table

```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     *
     * hashMap 基础的属性。一个数组，用于承载 node，table 的每一个格被称为桶
     */
    transient Node<K,V>[] table;
```

### Node

```java
	/**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     *
     * hashMap 中基础的 node 节点，用于存储 key, value
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
    }
```

### modCount

这个属性与理解 HashMap 的核心流程无关，如果读者只关心核心流程，可以不用关注。

```java
    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     *
     * 用于记录修改次数，每次增删改时会维护它的值
     * 在一个迭代器开始的时候，会把 modCount 用一个局部变量 mc 记录下来。迭代器遍历完成后，如果发现 modCount 和 mc 不相同，说明迭代期间 hashMap 进行过修改，则抛出异常。
     * 关于迭代器遍历，可以看一下 EntrySet 内部类的 forEach 方法
     */
    transient int modCount;
```

关于迭代器遍历，可以看一下 EntrySet 内部类的 forEach 方法。

```java
    /**
     * 请注意，源码里 EntrySet 的其他成员属性和成员方法，这里不作展示
     */
	final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                // 把 modCount 用一个局部变量 mc 记录下来
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                // 迭代器遍历完成后，如果发现 modCount 和 mc 不相同，说明迭代期间 hashMap 进行过修改，则抛出异常。
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
```

### threshold 扩容阈值

```java
    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * 扩容阈值，由 capacity * loadFactor 得到。决定何时 hashMap 执行 resize 方法扩容
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;
```

### loadFactor 加载因子

```java
    /**
     * The load factor for the hash table.
     *
     * 加载因子，决定了 hashMap 实际能存储的元素容量
     *
     * @serial
     */
    final float loadFactor;

	/**
     * The load factor used when none specified in constructor.
     *
     * 默认的 loadFactor 加载因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

加载因子是表示 HashMap 中元素的填满的程度。加载因子的目的是，为了降低 HashMap 中的 hash 冲突几率，防止大量 node 都因为 hash 冲突变成了链表或树，同时平衡占用的空间开销。

加载因子越大，填满的元素越多。优点是，空间利用率高了。缺点是，hash 冲突的机会加大了。

加载因子越小，填满的元素越少。优点是，冲突的机会减小了。缺点是，空间浪费多了。

默认的加载因子 DEFAULT_LOAD_FACTOR = 0.75f 算是在 hash 冲突几率与空间开销间做了取舍平衡。



## 构造方法

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity. 初始容量，由 capacity * loadFactor 可以得到扩容阈值 threshold
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

我们可以注意到，HashMap 里并没有 capacity 这个属性，我们在构造方法中传入的 capacity，其实会经过 capacity * loadFactor 计算，得到扩容阈值 threshold。



## put 方法

```java
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * put 方法，调用 Val 给指定的 key 添加对应的 value
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        // 第一个 boolean false 表示：当要 put 的 key 在 hashMap 中已存在时，会直接覆盖原有 value。第二个 boolean true 不用关心，与主流程无关。
        return putVal(hash(key), key, value, false, true);
    }
```

### putVal

```java
    /**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value  如果目标 key 在 hashMap 中已经存在，则不会覆盖原有的 value
     * @param evict if false, the table is in creation mode. 不用关心，与主流程无关
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 如果 table 还没有初始化，则初始化 table
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 桶位置计算公式: (n - 1) & hash。如果定位到的桶位置为空，则把 node 插入桶位置。p 指向桶位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        // 桶位置不为空，说明出现 hash 碰撞，走 else 分支
        else {
            Node<K,V> e; K k;
            // 通过 key 的 hash 值和 equals 方法判断桶位置上的 key 是否相同。如果相同则用 e 指向这个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 判断桶位置上是否是一棵树，如果是一棵树，则调用树添加元素的方法，然后用 e 指向树上的这个节点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 不是树，则说明是链表
            else {
                // 迭代链表，binCount 是链表长度计数
                for (int binCount = 0; ; ++binCount) {
                    // 用 e 指向本次迭代的当前元素。如果本次迭代，当前元素为空，即到达了链表的尾部
                    if ((e = p.next) == null) {
                        // 向链表尾部追加 node
                        p.next = newNode(hash, key, value, null);
                        // 如果链表长度达到了阈值，把链表转换成树。链表转换树的阈值无法修改，因为是 final 修饰的。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 通过 key 的 hash 值和 equals 方法判断本次迭代的 key 是否相同。如果相同则用 e 指向这个节点。然后停止迭代链表。
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    // 维护当前元素，准备下一次迭代
                    p = e;
                }
            }
            // 如果 e 不为空，说明要添加的 key 原先已存在于这个桶位置上，覆盖原有 value。这里体现了 hashMap 的 onlyIfAbsent 选项为 false 时，出现 key 相同时，会直接覆盖原有 value
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // onlyIfAbsent 选项为 false 时，或原有 value 为 null 时，会直接覆盖原有 value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 这个方法不用关心。留给 LinkedHashMap 回调用。
                afterNodeAccess(e);
                // 返回原有的 value
                return oldValue;
            }
        }
        // 维护修改次数计数
        ++modCount;
        // 如果达到了扩容阈值，则 resize
        if (++size > threshold)
            resize();
        // 这个方法不用关心。留给 LinkedHashMap 回调用。
        afterNodeInsertion(evict);
        // 没有找到 key 对应的 value，返回 null
        return null;
    }
```

有一个细节，链表转换树的阈值 TREEIFY_THRESHOLD 无法修改，因为是 final 修饰的，之前面试被问到过。

```java

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     *
     * 链表转换树的阈值，无法修改，因为是 final 修饰的
     */
    static final int TREEIFY_THRESHOLD = 8;
```



## get 方法

```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        // 根据指定的 key 查找 node，返回 node 的 value
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

### getNode

```java
    /**
     * Implements Map.get and related methods.
     * 
     * 根据指定的 key，查找 node
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 如果 table 不为空，且根据 key 对应到的桶位置不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 通过 key 的 hash 值和 equals 方法判断桶位置上的 key 是否相同
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                // 和目标 key 相同，返回当前节点
                return first;
            // 桶位置不为空，说明可能存在 hash 碰撞，判断桶位置上的元素是否有下一个节点
            if ((e = first.next) != null) {
                // 判断桶位置上是否是一棵树，如果是一棵树，则调用树查找元素的方法
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 不是树，则说明是链表，迭代链表
                do {
                    // 通过 key 的 hash 值和 equals 方法判断本次迭代的 key 是否相同
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 和目标 key 相同，返回当前节点
                        return e;
                } while ((e = e.next) != null);
            }
        }
        // 没有找到 key 对应的元素，返回 null
        return null;
    }
```



## resize 方法

### 常见的执行 resize() 方法的两种情况

在 HashMap 的 putVal 方法中，如果 table 未初始化，则会执行resize()，然后就初始化table。初始化 table 由 resize 负责。

```java
// 如果 table 还没有初始化，则初始化 table        
if ((tab = table) == null || (n = tab.length) == 0)
	n = (tab = resize()).length;
```

在 HashMap 的 putVal 方法中，存储的数据量大于 threshold 时，会执行 resize() 方法。

```java
        // 如果 hashMap 中的元素数量达到了扩容阈值，则 resize
        if (++size > threshold)
            resize();
```

在putVal()方法中，size表示当前HashMap的数据量，如果size大于threshold,则会执行该方法，进行扩容操作。

### resize 方法源码

```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * 初始化 hashMap 或给 hashMap 的 table 扩容两倍
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        // oldTab 指向旧 table
        Node<K,V>[] oldTab = table;
        // 旧 table 的长度
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // oldThr 表示旧的扩容阈值 threshold。threshold = 数组长度 * 负载因子
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 当旧 table 的长度大于最大容量时的处理
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 如果旧的数组长度 * 2 后小于 int 的最大值，并且旧的数组长度大于 16
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                // 扩容阈值 * 2
                newThr = oldThr << 1; // double threshold
        }
        // 如果旧的 threshold 大于 0，初始容量设置为旧的 threshold。这里在 table 初始化时会用到
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        // 扩容阈值为 0 表示使用默认值，DEFAULT_INITIAL_CAPACITY = 16，DEFAULT_LOAD_FACTOR = 0.75，因此默认的扩容阈值为 12
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 扩容阈值为 0 时的边界条件处理
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        // 将计算后得出的阈值赋值给 threshold 属性
        threshold = newThr;
        // 不用关心这个注解。这个注解在屏蔽一些无关紧要的警告，使开发者能看到一些他们真正关心的警告，降低开发者的心智负担。
        // 创建一个新的 table，供扩容后使用
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 把新 table 赋值给 hashMap 的属性
        table = newTab;
        // 如果旧 table 不为空，开始扩容
        if (oldTab != null) {
            // 迭代遍历旧 table
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                // e 指向当前桶位置的元 node，当前桶位置的 node 不为空时
                if ((e = oldTab[j]) != null) {
                    // 清空旧 table 的当前桶位置
                    oldTab[j] = null;
                    if (e.next == null)
                        // 如果当前桶位置的 node 不是链表不是红黑树，则根据桶位置计算公式，重新分配 node 的桶位置
                        newTab[e.hash & (newCap - 1)] = e;
                    // 如果当前桶位置的 node 是树，则使用树的方式，把旧树上的 node，重新分配到新的树中
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 不是树，则是链表
                    else { // preserve order
                        // 把链表中的所有节点分成两条链表
                        // 一条链表的 node 是不需要更换 table 下标的
                        Node<K,V> loHead = null, loTail = null;
                        // 一条链表的 node 是需要更换 table 下标的
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        // 迭代遍历链表
                        do {
                            next = e.next;
                            // 如果 e.hash & oldCap 进行二进制与运算，算出的结果为 0，即说明该 node 所对应的数组下标不需要改变。把该 node 追加到 loHead 链表上
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 否则说明该 node 所对应的数组下标需要改变。把该 node 追加到 hiHead 链表上
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        // 如果不需要更换 table 下标的 node 链表 -- loTail 不为空，则把 loTail 放在当前桶位置上
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 如果需要更换 table 下标的 node 链表 -- hiTail 不为空，则把 hiTail 放到新的桶位置上。并且计算公式是把当前 table 下标直接 + 旧 table 的长度
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        // 返回新创建的 table
        return newTab;
    }
```

#### 两个个公式

##### (e.hash & oldCap) == 0 判断是否需要重新分配桶位置

e 是当前 node，oldCap 是旧数组的长度。这个公式算出的结果为 0，说明该 node (即 e)所对应的数组下标不需要改变。结果不为 0，说明该 node 所对应的数组下标需要改变。

`(e.hash & oldCap) == 0` 为什么能判断出是否需要重新分配桶位置？

这个公式是推导出来的，推导过程是数学，我们不需要关注。如果想了解此公式的推导请见：[HashMap扩容时的rehash方法中(e.hash & oldCap) == 0算法推导](https://blog.csdn.net/u010425839/article/details/106620440)

##### j + oldCap 桶位置重分配公式

j 是 node 的旧桶位置，oldCap 是旧 table 的长度。即 旧桶位置 + 旧 table 的长度。得到这个公式的运算结果，是扩容后该元素的新桶位置。可以理解为是桶位置重新分配的公式。

为什么这样能得到呢？我们来举例回答一下。

现在我们有一个 node key 的 hash 值是 9，对应的二进制位。旧 table 的长度 oldCap 是 8，新 table 的长度 newCap 是 16。以下是手写演算验证：

![桶位置重新分配公式的验证](https://image-hosting.jellyfishmix.com/20220729192721.jpg)

#### 为什么 HashMap 扩容是 2 倍？

通过本文桶位置重新分配的公式 `j + oldCap` 手写验证，我们可以看出，当 HashMap 扩容两倍的时候，刚好可以用到 桶位置重新分配的公式 `j + oldCap`，加快计算重分配后的桶位置。同时，`newCap = oldCap << 1` 新 table 长度 = 旧 table 长度在二进制上左移一位，这样的位运算也很高效。

其实这里扩容倍数和桶位置重分配公式的配合，能体现出作者缜密的思考和深厚的数学功底。