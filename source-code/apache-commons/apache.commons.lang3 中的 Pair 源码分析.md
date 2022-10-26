# apache.commons.lang3 中的 Pair 源码分析



## 说明

1. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 介绍

Pair 用于存储左右元素对，Pair<Left, Right>。主要有两个子类，ImmutablePair 和 MutablePair。



## Pair

org.apache.commons.lang3.tuple.Pair

### 类签名

```java
/**
 * A pair consisting of two elements.
 *
 * <p>This class is an abstract implementation defining the basic API.
 * It refers to the elements as 'left' and 'right'. It also implements the
 * {@code Map.Entry} interface where the key is 'left' and the value is 'right'.</p>
 *
 * <p>Subclass implementations may be mutable or immutable.
 * However, there is no restriction on the type of the stored objects that may be stored.
 * If mutable objects are stored in the pair, then the pair itself effectively becomes mutable.</p>
 *
 * Pair 用于存储左右元素对，Pair<Left, Right>。主要有两个子类，ImmutablePair 和 MutablePair。
 * 实现了 Map.Entry, Comparable, Serializable
 *
 * @param <L> the left element type
 * @param <R> the right element type
 *
 * @since 3.0
 */
public abstract class Pair<L, R> implements Map.Entry<L, R>, Comparable<Pair<L, R>>, Serializable
```

注意 Pair 实现了 java.util.Map.Entry，因此 left 和 right 也可以视作 key-value。

### getLeft, getRight 方法

获取左右元素 left, right, 抽象方法，留给子类实现。

```java
    /**
     * Gets the left element from this pair.
     *
     * 抽象方法，留给子类实现
     *
     * <p>When treated as a key-value pair, this is the key.</p>
     *
     * @return the left element, may be null
     */
    public abstract L getLeft();

    /**
     * Gets the right element from this pair.
     *
     * 抽象方法，留给子类实现
     *
     * <p>When treated as a key-value pair, this is the value.</p>
     *
     * @return the right element, may be null
     */
    public abstract R getRight();
```

### getKey, getValue 方法

因为 Pair 实现了 java.util.Map.Entry，因此需要实现父接口的 getKey() 和 getValue() 方法。获取 key, value, 本质上在调用 getLeft() 方法和 getRight() 方法。

```java
	/**
     * Gets the key from this pair.
     *
     * <p>This method implements the {@code Map.Entry} interface returning the
     * left element as the key.</p>
     *
     * 实现 java.util.Map.Entry#getKey() 方法，调用 getLeft() 方法
     *
     * @return the left element as the key, may be null
     */
    @Override
    public final L getKey() {
        return getLeft();
    }    

	/**
     * Gets the value from this pair.
     *
     * <p>This method implements the {@code Map.Entry} interface returning the
     * right element as the value.</p>
     *
     * 实现 java.util.Map.Entry#getValue() 方法，调用 getRight() 方法
     *
     * @return the right element as the value, may be null
     */
    @Override
    public R getValue() {
        return getRight();
    }
```

### of 方法--以 static 调用的形式创建一个 Pair 实例

```java
	/**
     * Creates an immutable pair of two objects inferring the generic types.
     *
     * <p>This factory allows the pair to be created using inference to
     * obtain the generic types.</p>
     *
     * of 方法，以 static 调用的形式创建一个 Pair 实例。默认创建的是 ImmutablePair。
     *
     * @param <L> the left element type
     * @param <R> the right element type
     * @param left  the left element, may be null
     * @param right  the right element, may be null
     * @return a pair formed from the two parameters, not null
     */
    public static <L, R> Pair<L, R> of(final L left, final R right) {
        return ImmutablePair.of(left, right);
    }
```

of 方法，以 static 调用的形式创建一个 Pair。默认创建的是 ImmutablePair。



## ImmutablePair

org.apache.commons.lang3.tuple.ImmutablePair

继承了 Pair，left 和 right 值不可变

### 类签名

```java
/**
 * An immutable pair consisting of two {@link Object} elements.
 *
 * <p>Although the implementation is immutable, there is no restriction on the objects
 * that may be stored. If mutable objects are stored in the pair, then the pair
 * itself effectively becomes mutable. The class is also {@code final}, so a subclass
 * can not add undesirable behavior.</p>
 *
 * <p>#ThreadSafe# if both paired objects are thread-safe</p>
 *
 * 继承了 Pair，left 和 right 值不可变
 *
 * @param <L> the left element type
 * @param <R> the right element type
 *
 * @since 3.0
 */
public class ImmutablePair<L, R> extends Pair<L, R>
```

### 左右元素 left, right

```java
    /**
     * Left object
     *
     * final 修饰，left 值第一次被设置后不可变
     */
    public final L left;

    /**
     * Right object
     *
     * final 修饰，right 值第一次被设置后不可变
     */
    public final R right;
```

final 修饰，left, right 值第一次被设置后不可变。

### setValue 方法

```java
    /**
     * Throws {@link UnsupportedOperationException}.
     *
     * <p>This pair is immutable, so this operation is not supported.</p>
     *
     * 因为 ImmutablePair 的 right 值不可变，所以如果 set 直接抛异常
     *
     * @param value  the value to set
     * @return never
     * @throws UnsupportedOperationException as this operation is not supported
     */
    @Override
    public R setValue(final R value) {
        throw new UnsupportedOperationException();
    }
```

因为 ImmutablePair 的 right 值不可变，所以如果 setValue 直接抛异常。



## MutablePair

org.apache.commons.lang3.tuple.MutablePair

继承了 Pair，left 和 right 值可变

### 类签名

```java
/**
 * A mutable pair consisting of two {@link Object} elements.
 *
 * <p>Not #ThreadSafe#</p>
 *
 * 继承了 Pair，left 和 right 值可变
 *
 * @param <L> the left element type
 * @param <R> the right element type
 *
 * @since 3.0
 */
public class MutablePair<L, R> extends Pair<L, R>
```

### 左右元素 left, right

```java
    /**
     * Left object
     *
     * 没有 final 修饰，left 值第一次被设置后可变
     */
    public L left;

    /**
     * Right object
     *
     * 没有 final 修饰，right 值第一次被设置后可变
     */
    public R right;
```

没有 final 修饰，left, right 值第一次被设置后可变。

### setValue 方法

```java
    /**
     * Sets the {@code Map.Entry} value.
     * This sets the right element of the pair.
     *
     * set value，本质上是设置 right。会返回旧 right 的值。
     *
     * @param value  the right value to set, not null
     * @return the old value for the right element
     */
    @Override
    public R setValue(final R value) {
        final R result = getRight();
        setRight(value);
        return result;
    }
```

set value，本质上是设置 right。会返回旧 right 的值。



## Pair 与 java.util.Map

1. Pair 和 Map 共通点: Pair 和 Map 都是以 key, value 进行存储。
2. Pair 和 Map 不同点:
   1. Pair 通过 getKey(), getValue() 获取 key 和 value，没有增加键值对的操作。
   2. Map 是通过 get(key) 获取 key 对应的 value，通过 values() 获取所有的value，而且还可以通过 put 增加键值对。
   3. Pair 保存的是一对 key-value，而 Map 可以保存多对 key-value。