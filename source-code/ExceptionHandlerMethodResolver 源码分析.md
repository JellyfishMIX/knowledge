# ExceptionHandlerMethodResolver 源码分析



## ExceptionHandler 的执行顺序

### getMappedMethod 方法

ExceptionHandlerMethodResolver 中的 getMappedMethod 方法，是决定哪个 ExceptionHandler 先执行的核心代码。

```java
	/**
	 * Return the {@link Method} mapped to the given exception type, or {@code null} if none.
	 *
	 * 决定哪个 ExceptionHandler 先执行的核心代码
	 */
	@Nullable
	private Method getMappedMethod(Class<? extends Throwable> exceptionType) {
		List<Class<? extends Throwable>> matches = new ArrayList<>();
		// 首先找到可以匹配异常的所有 ExceptionHandler
		for (Class<? extends Throwable> mappedException : this.mappedMethods.keySet()) {
			if (mappedException.isAssignableFrom(exceptionType)) {
				matches.add(mappedException);
			}
		}
		if (!matches.isEmpty()) {
			// 根据深度(异常匹配度)做排序
			matches.sort(new ExceptionDepthComparator(exceptionType));
			// 取深度最小的那个 handler(即匹配度最高的那个)
			return this.mappedMethods.get(matches.get(0));
		}
		else {
			return null;
		}
	}
```

可以看出，首先找到可以匹配异常的所有 ExceptionHandler，然后根据深度(异常匹配度)做排序，取深度最小的那个 handler(即匹配度最高的那个)。根据深度(异常匹配度)做排序，使用的是 ExceptionDepthComparator，构造一个 exceptionType 深度比较器。

### ExceptionDepthComparator

```java
/**
 * Comparator capable of sorting exceptions based on their depth from the thrown exception type.
 * 
 * exceptionType 深度比较器
 *
 * @author Juergen Hoeller
 * @author Arjen Poutsma
 * @since 3.0.3
 */
public class ExceptionDepthComparator implements Comparator<Class<? extends Throwable>> {

	/**
	 * 目标异常(与此异常做比较)
	 */
	private final Class<? extends Throwable> targetException;

	/**
	 * Create a new ExceptionDepthComparator for the given exception type.
	 *
	 * 构造一个 exceptionType 深度比较器
	 *
	 * @param exceptionType the target exception type to compare to when sorting by depth
	 */
	public ExceptionDepthComparator(Class<? extends Throwable> exceptionType) {
		Assert.notNull(exceptionType, "Target exception type must not be null");
		this.targetException = exceptionType;
	}

	/**
	 * 比较 o1 和 o2 谁的深度更小，深度小的排在前面
	 *
	 * @param o1
	 * @param o2
	 * @return
	 */
	@Override
	public int compare(Class<? extends Throwable> o1, Class<? extends Throwable> o2) {
		int depth1 = getDepth(o1, this.targetException, 0);
		int depth2 = getDepth(o2, this.targetException, 0);
		return (depth1 - depth2);
	}

	/**
	 * exceptionToMatch 是 declaredException 或 declaredException 的子类
	 * 递归地计算 exceptionToMatch 距离 declaredException 的深度(即隔着多少个层级)
	 *
	 * @param declaredException
	 * @param exceptionToMatch
	 * @param depth
	 * @return
	 */
	private int getDepth(Class<?> declaredException, Class<?> exceptionToMatch, int depth) {
		if (exceptionToMatch.equals(declaredException)) {
			// Found it!
			return depth;
		}
		// If we've gone as far as we can go and haven't found it...
		if (exceptionToMatch == Throwable.class) {
			return Integer.MAX_VALUE;
		}
		return getDepth(declaredException, exceptionToMatch.getSuperclass(), depth + 1);
	}
}
```

可以看到 `ExceptionDepthComparator implements Comparator`，经典的比较器实现方式。规则是深度小的 exceptionType 排在前面。计算 exceptionType 深度采用的方法是，递归地计算 exceptionToMatch 距离 declaredException 的深度(即隔着多少个层级)。



## 未完待续

项目中需要思考 ExceptionHandler 的执行顺序，时间关系目前先分析这一点，ExceptionHandlerMethodResolver 中其他内容日后补充。