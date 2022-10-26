# Optional 源码分析



## 前言

1. 本文基于 jdk 8 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

类签名：

```java
/**
 * A container object which may or may not contain a non-null value.
 * If a value is present, {@code isPresent()} will return {@code true} and
 * {@code get()} will return the value.
 *
 * <p>Additional methods that depend on the presence or absence of a contained
 * value are provided, such as {@link #orElse(java.lang.Object) orElse()}
 * (return a default value if value not present) and
 * {@link #ifPresent(java.util.function.Consumer) ifPresent()} (execute a block
 * of code if the value is present).
 *
 * <p>This is a <a href="../lang/doc-files/ValueBased.html">value-based</a>
 * class; use of identity-sensitive operations (including reference equality
 * ({@code ==}), identity hash code, or synchronization) on instances of
 * {@code Optional} may have unpredictable results and should be avoided.
 *
 * @since 1.8
 */
public final class Optional<T>
```



## 关键属性

```java
    /**
     * Common instance for {@code empty()}.
     *
     * 调用了无参构造方法，创建一个不可变的 Optional 对象，名为 EMPTY。这个对象的属性 value 为 null。
     * EMPTY 大写体现了这是一个不可变对象(常量)
     */
    private static final Optional<?> EMPTY = new Optional<>();

    /**
     * If non-null, the value; if null, indicates no value is present
     *
     * 用于存储传入的泛型对象，如果空，则其值为 null
     */
    private final T value;
```



## 构造方法

```java
	/**
     * Constructs an empty instance.
     *
     * 无参构造方法，value 属性为 null
     * private 修饰，意味着此方法只供 Optional 类内部使用，无法通过此构造方法直接创建 Optional 对象。
     *
     * @implNote Generally only one empty instance, {@link Optional#EMPTY},
     * should exist per VM.
     */
    private Optional() {
        this.value = null;
    }

    /**
     * Constructs an instance with the value present.
     *
     * 有参构造方法，value 属性为传入的值，要求 value 必须不为 null，否则抛出 NullPointerException
     * private 修饰，意味着此方法只供 Optional 类内部使用，无法通过此构造方法直接创建 Optional 对象。
     *
     * @param value the non-null value to be present
     * @throws NullPointerException if value is null
     */
    private Optional(T value) {
        this.value = Objects.requireNonNull(value);
    }
```

无参构造方法，value 属性为 null。

有参构造方法，value 属性为传入的值，要求 value 必须不为 null，否则抛出 NullPointerException。

Optional 的构造方法均有 private 修饰，意味着 Optional 的构造方法，只供 Optional 类内部使用。在类外部，无法通过这些构造方法直接创建 Optional 对象。



## 工厂方法

### empty 方法

```java
    /**
     * Returns an empty {@code Optional} instance.  No value is present for this
     * Optional.
     *
     * 返回一个 value 为 null 的 Optional 对象，这个对象是可变的。不是返回 EMPTY 那个不可变对象。
     *
     * @apiNote Though it may be tempting to do so, avoid testing if an object
     * is empty by comparing with {@code ==} against instances returned by
     * {@code Option.empty()}. There is no guarantee that it is a singleton.
     * Instead, use {@link #isPresent()}.
     *
     * @param <T> Type of the non-existent value
     * @return an empty {@code Optional}
     */
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }
```

返回一个 value 为 null 的 Optional 对象，这个对象是可变的。不是返回 EMPTY 那个不可变对象。

### of 方法

```java
	/**
     * Returns an {@code Optional} with the specified present non-null value.
     *
     * of 静态方法是 Optional 的静态工厂方法，用于返回 Optional 的实例。要求传入的 value 必须不为 null，否则抛出 NullPointerException
     * 请注意，Optional 的构造方法都是 private 修饰的，无法通过构造方法创建 Optional 对象。需要借助 of 这样的工厂方法来创建实例。
     *
     * @param <T> the class of the value
     * @param value the value to be present, which must be non-null
     * @return an {@code Optional} with the value present
     * @throws NullPointerException if value is null
     */
    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }
```

of 静态方法是 Optional 的静态工厂方法，用于返回 Optional 的实例。要求传入的 value 必须不为 null，否则抛出 NullPointerException。

请注意，Optional 的构造方法都是 private 修饰的，无法通过构造方法创建 Optional 对象。需要借助 of 这样的工厂方法来创建实例。

### ofNullable 方法

```java
	/**
     * Returns an {@code Optional} describing the specified value, if non-null,
     * otherwise returns an empty {@code Optional}.
     *
     * 允许输入参数为 null 的 of 工厂方法。当 value 为 null 时，调用 empty() 方法创建一个属性 value 为 null 的 Optional 对象。
     * 否则，调用有参构造方法，把 value 作为入参传入。
     *
     * @param <T> the class of the value
     * @param value the possibly-null value to describe
     * @return an {@code Optional} with a present value if the specified value
     * is non-null, otherwise an empty {@code Optional}
     */
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
```

允许输入参数为 null 的 of 工厂方法。当 value 为 null 时，调用 empty() 方法创建一个属性 value 为 null 的 Optional 对象。

否则，调用有参构造方法，把 value 作为入参传入。



## get 方法

```java
	/**
     * If a value is present in this {@code Optional}, returns the value,
     * otherwise throws {@code NoSuchElementException}.
     *
     * get 方法作为一个非静态方法，通常是被使用 of/ofNullable 静态工厂方法的 Optional 对象所调用，返回其属性 value。
     *
     * api 使用 demo:
     * Optional.ofNullable(object).get()
     *
     * @return the non-null value held by this {@code Optional}
     * @throws NoSuchElementException if there is no value present
     *
     * @see Optional#isPresent()
     */
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }
```

get 方法作为一个非静态方法，通常是被使用 of/ofNullable 静态工厂方法的 Optional 对象所调用，返回其属性 value。

api 使用 demo: `Optional.ofNullable(object).get()`



## isPresent 方法

```java
	/**
     * Return {@code true} if there is a value present, otherwise {@code false}.
     *
     * 返回 boolean 值，用于判断 optional 对象的属性 value 值是否为 null
     *
     * @return {@code true} if there is a value present, otherwise {@code false}
     */
    public boolean isPresent() {
        return value != null;
    }
```

返回 boolean 值，用于判断 optional 对象的属性 value 值是否为 null



## ifPresent 方法

```java
    /**
     * If a value is present, invoke the specified consumer with the value,
     * otherwise do nothing.
     *
     * 如果 optional 对象的属性 value 不为 null，则使用该 value 值作为入参，调用实现 Consumer 接口的 lambda 回调函数。否则，不做任何事情。
     * consumer 是传入的 lambda 回调函数
     *
     * @param consumer block to be executed if a value is present
     * @throws NullPointerException if value is present and {@code consumer} is
     * null
     */
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```

如果 optional 对象的属性 value 不为 null，则使用该 value 值作为入参，调用实现 Consumer 接口的 lambda 回调函数。否则，不做任何事情。

consumer 是传入的 lambda 回调函数。



## filter 方法

```java
	/**
     * If a value is present, and the value matches the given predicate,
     * return an {@code Optional} describing the value, otherwise return an
     * empty {@code Optional}.
     *
     * 首先，入参 lambda 不能为 null，否则抛出 NullPointerException。
     * 如果 optional 对象的属性 value 为 null，则返回当前 optional 对象本身。
     * 否则，optional 对象的属性 value 作为入参，调用传入的 lambda 回调函数判断是否要过滤。
     * 如果判断结果为 true 则放行，返回当前 optional 对象本身。
     * 如果判断结果为 false 则过滤，返回一个新构建的 value 为 null 的 optional 对象。
     *
     * @param predicate a predicate to apply to the value, if present
     * @return an {@code Optional} describing the value of this {@code Optional}
     * if a value is present and the value matches the given predicate,
     * otherwise an empty {@code Optional}
     * @throws NullPointerException if the predicate is null
     */
    public Optional<T> filter(Predicate<? super T> predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
    }
```

1. 首先，入参 lambda 不能为 null，否则抛出 NullPointerException。

2. 如果 optional 对象的属性 value 为 null，则返回当前 optional 对象本身。

3.  否则，optional 对象的属性 value 作为入参，调用传入的 lambda 回调函数判断是否要过滤。

4. 如果判断结果为 true 则放行，返回当前 optional 对象本身。

5. 如果判断结果为 false 则过滤，返回一个新构建的 value 为 null 的 optional 对象。



## map 方法

```java
    /**
     * If a value is present, apply the provided mapping function to it,
     * and if the result is non-null, return an {@code Optional} describing the
     * result.  Otherwise return an empty {@code Optional}.
     *
     * @apiNote This method supports post-processing on optional values, without
     * the need to explicitly check for a return status.  For example, the
     * following code traverses a stream of file names, selects one that has
     * not yet been processed, and then opens that file, returning an
     * {@code Optional<FileInputStream>}:
     *
     * <pre>{@code
     *     Optional<FileInputStream> fis =
     *         names.stream().filter(name -> !isProcessedYet(name))
     *                       .findFirst()
     *                       .map(name -> new FileInputStream(name));
     * }</pre>
     *
     * Here, {@code findFirst} returns an {@code Optional<String>}, and then
     * {@code map} returns an {@code Optional<FileInputStream>} for the desired
     * file if one exists.
     *
     * 首先，入参 lambda 不能为 null，否则抛出 NullPointerException
     * 如果 optional 对象的属性 value 为 null，则返回一个 value 为 null 的 optional 对象
     * 否则，把 value 作为入参，调用 lambda 回调函数，得到返回值。
     * lambda 回调函数返回值传入 Optional.ofNullable() 静态工厂方法，得到创建出的新 optional 对象。
     *
     * @param <U> The type of the result of the mapping function
     * @param mapper a mapping function to apply to the value, if present
     * @return an {@code Optional} describing the result of applying a mapping
     * function to the value of this {@code Optional}, if a value is present,
     * otherwise an empty {@code Optional}
     * @throws NullPointerException if the mapping function is null
     */
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }
```

1. 首先，入参 lambda 不能为 null，否则抛出 NullPointerException
2. 如果 optional 对象的属性 value 为 null，则返回一个 value 为 null 的 optional 对象
3. 否则，把 value 作为入参，调用 lambda 回调函数，得到返回值。
4. lambda 回调函数返回值传入 Optional.ofNullable() 静态工厂方法，得到创建出的新 optional 对象。



## flatMap 方法

```java
	/**
     * If a value is present, apply the provided {@code Optional}-bearing
     * mapping function to it, return that result, otherwise return an empty
     * {@code Optional}.  This method is similar to {@link #map(Function)},
     * but the provided mapper is one whose result is already an {@code Optional},
     * and if invoked, {@code flatMap} does not wrap it with an additional
     * {@code Optional}.
     *
     * 首先，入参 lambda 不能为 null，否则抛出 NullPointerException
     * 如果 optional 对象的属性 value 为 null，则返回一个 value 为 null 的 optional 对象
     * 否则，把 value 作为入参，调用 lambda 回调函数，得到返回值。
     * lambda 回调函数返回值需要是 Optional 类的实例，如果返回值为 null，会抛出 NullPointerException
     *
     * @param <U> The type parameter to the {@code Optional} returned by
     * @param mapper a mapping function to apply to the value, if present
     *           the mapping function
     * @return the result of applying an {@code Optional}-bearing mapping
     * function to the value of this {@code Optional}, if a value is present,
     * otherwise an empty {@code Optional}
     * @throws NullPointerException if the mapping function is null or returns
     * a null result
     */
    public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Objects.requireNonNull(mapper.apply(value));
        }
    }
```

1. 首先，入参 lambda 不能为 null，否则抛出 NullPointerException
2. 如果 optional 对象的属性 value 为 null，则返回一个 value 为 null 的 optional 对象
3. 否则，把 value 作为入参，调用 lambda 回调函数，得到返回值。
4. lambda 回调函数返回值需要是 Optional 类的实例，如果返回值为 null，会抛出 NullPointerException



## orElse 方法

```java
    /**
     * Return the value if present, otherwise return {@code other}.
     *
     * 如果 optional 对象的 value 不为 null，则返回 value。否则返回入参 other。
     * 可以理解为，入参 other 是当 optional 对象属性 value 为 null 时的默认值。
     *
     * @param other the value to be returned if there is no value present, may
     * be null
     * @return the value, if present, otherwise {@code other}
     */
    public T orElse(T other) {
        return value != null ? value : other;
    }
```

1. 如果 optional 对象的 value 不为 null，则返回 value。否则返回入参 other。
2. 可以理解为，入参 other 是当 optional 对象属性 value 为 null 时的默认值。



## orElseGet 方法

```java
	/**
     * Return the value if present, otherwise invoke {@code other} and return
     * the result of that invocation.
     *
     * 如果 optional 对象的 value 不为 null，则返回 value。否则执行入参 lambda 回调函数得到返回值。
     * 可以理解为，入参 other 这个 lambda 回调函数，在 optional 对象属性为 null 时会触发执行，并返回 lambda 回调函数的返回值。
     *
     * @param other a {@code Supplier} whose result is returned if no value
     * is present
     * @return the value if present otherwise the result of {@code other.get()}
     * @throws NullPointerException if value is not present and {@code other} is
     * null
     */
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
```

1. 如果 optional 对象的 value 不为 null，则返回 value。否则执行入参 lambda 回调函数得到返回值。
2. 可以理解为，入参 other 这个 lambda 回调函数，在 optional 对象属性为 null 时会触发执行，并返回 lambda 回调函数的返回值。