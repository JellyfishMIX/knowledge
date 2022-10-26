# AliasRegistry 源码分析



## 前言

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

向上继承、实现关系：

![image-20220817112233328](https://image-hosting.jellyfishmix.com/20220817112233.png)

向上向下所属层次：

![image-20220817121442405](https://image-hosting.jellyfishmix.com/20220817121442.png)



## 什么是别名？

AliasRegistry 声明了管理别名的一些方法，我们先来看一下什么是别名。

别名 -> 本名，一个本名可以对应多个别名，如成龙有别名元楼、陈元龙，那么映射关系就是
* 元楼 -> 成龙
* 陈元龙 -> 成龙

### 别名的使用 demo

AliasRegistry 在 spring 中只有一个直接实现类 SimpleAliasRegistry。先看一下 SimpleAliasRegistry 的简单使用。

```java
public class Client {
  public static void main(String[] args) {
    AliasRegistry aliasRegistry = new SimpleAliasRegistry();
    aliasRegistry.registerAlias("userService", "userService1");
    aliasRegistry.registerAlias("userService", "userService2");
      
    System.out.println("userService 的别名为 " + Arrays.toString(aliasRegistry.getAliases("userService")));
    System.out.println("userService1 是否为别名：" + aliasRegistry.isAlias("userService1"));

    aliasRegistry.removeAlias("userService1");
    System.out.println("userService 的别名为 " + Arrays.toString(aliasRegistry.getAliases("userService")));
    System.out.println("userService1 是否为别名：" + aliasRegistry.isAlias("userService1"));
  }
}
```

对 userService 注册两个别名 userService1，userService2，输出结果为：

```
userService 的别名为 [userService2,userService1]
userService1 是否为别名：true
userService 的别名为 [userService2]
userService1 是否为别名：false
```



## 源码分析

```java
/**
 * Common interface for managing aliases. Serves as a super-interface for
 * {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}.
 *
 * 作为 IOC 容器相关的最顶层接口之一，这个接口声明了管理别名的一些方法
 * 主要作用是将名字-别名映射存到内存中，提供查找和校验的接口
 *
 * @author Juergen Hoeller
 * @since 2.5.2
 */
public interface AliasRegistry {

	/**
	 * Given a name, register an alias for it.
	 *
	 * 为一个 name 注册别名
	 *
	 * @param name the canonical name
	 * @param alias the alias to be registered
	 * @throws IllegalStateException if the alias is already in use
	 * and may not be overridden
	 */
	void registerAlias(String name, String alias);

	/**
	 * Remove the specified alias from this registry.
	 *
	 * 删除一个指定的别名
	 *
	 * @param alias the alias to remove
	 * @throws IllegalStateException if no such alias was found
	 */
	void removeAlias(String alias);

	/**
	 * Determine whether the given name is defined as an alias
	 * (as opposed to the name of an actually registered component).
	 *
	 * 确定一个名字是否是别名
	 *
	 * @param name the name to check
	 * @return whether the given name is an alias
	 */
	boolean isAlias(String name);

	/**
	 * Return the aliases for the given name, if defined.
	 *
	 * 返回一个名字注册的别名列表
	 *
	 * @param name the name to check for aliases
	 * @return the aliases, or an empty array if none
	 */
	String[] getAliases(String name);

}
```

作为 IOC 容器相关的最顶层接口之一，这个接口声明了管理别名的一些方法。主要作用是将名字-别名映射存到内存中，提供查找和校验的接口。