# ConfigurableBeanFactory 源码分析



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

向上继承、实现关系：

![image-20220923025106715](https://image-hosting.jellyfishmix.com/20220923025106.png)

向上向下所属层次：

![image-20220923025256453](https://image-hosting.jellyfishmix.com/20220923025256.png)

类签名：

```java
/**
 * Configuration interface to be implemented by most bean factories. Provides
 * facilities to configure a bean factory, in addition to the bean factory
 * client methods in the {@link org.springframework.beans.factory.BeanFactory}
 * interface.
 *
 * <p>This bean factory interface is not meant to be used in normal application
 * code: Stick to {@link org.springframework.beans.factory.BeanFactory} or
 * {@link org.springframework.beans.factory.ListableBeanFactory} for typical
 * needs. This extended interface is just meant to allow for framework-internal
 * plug'n'play and for special access to bean factory configuration methods.
 *
 * @author Juergen Hoeller
 * @since 03.11.2003
 * @see org.springframework.beans.factory.BeanFactory
 * @see org.springframework.beans.factory.ListableBeanFactory
 * @see ConfigurableListableBeanFactory
 */
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory, SingletonBeanRegistry {
```



## 介绍

1. 可配置的 BeanFactory，声明了对 BeanFactory 各种各样的配置能力，如 bean 的作用域，bean 的 classLoader，bean 的元数据缓存，bean 的表达式解析器，类型转换器，属性编辑器等。实现此接口即拥有对 BeanFactory 的配置能力。
2. ConfigurableBeanFactory 这个巨大的 BeanFactory 体系接口，继承自 HierarchicalBeanFactory 和 SingletonBeanRegistry 这两个接口，并额外独有多个方法。
3. 本文介绍 ConfigurableBeanFactory 接口独有的主要方法。



## 属性

两个属性，singleton 表示原型模式。prototype 表示原型模式。

```java
	/**
	 * Scope identifier for the standard singleton scope: {@value}.
	 * <p>Custom scopes can be added via {@code registerScope}.
	 *
	 * 单例模式
	 *
	 * @see #registerScope
	 */
	String SCOPE_SINGLETON = "singleton";

	/**
	 * Scope identifier for the standard prototype scope: {@value}.
	 * <p>Custom scopes can be added via {@code registerScope}.
	 *
	 * 原型模式
	 *
	 * @see #registerScope
	 */
	String SCOPE_PROTOTYPE = "prototype";
```



## 父容器

setParentBeanFactory 方法设置父类容器。

```java
	/**
	 * Set the parent of this bean factory.
	 * <p>Note that the parent cannot be changed: It should only be set outside
	 * a constructor if it isn't available at the time of factory instantiation.
	 *
	 * 设置父类容器
	 *
	 * @param parentBeanFactory the parent BeanFactory
	 * @throws IllegalStateException if this factory is already associated with
	 * a parent BeanFactory
	 * @see #getParentBeanFactory()
	 */
	void setParentBeanFactory(BeanFactory parentBeanFactory) throws IllegalStateException;
```



## 类加载器

1. setBeanClassLoader 方法: 设置 bean 的类加载器。
2. getBeanClassLoader 方法: 获取 bean 的类加载器。
3. setTempClassLoader 方法: 设置临时加载器。如果涉及到加载时 aop 编织，通常仅指定一个临时类装入器，以确保实际的 bean 类被尽可能延迟地装入。一旦 BeanFactory 完成他的启动解析后，这个临时的类加载器将被移除。
4. getTempClassLoader 方法: 获取临时加载器。

```java
	/**
	 * Set the class loader to use for loading bean classes.
	 * Default is the thread context class loader.
	 * <p>Note that this class loader will only apply to bean definitions
	 * that do not carry a resolved bean class yet. This is the case as of
	 * Spring 2.0 by default: Bean definitions only carry bean class names,
	 * to be resolved once the factory processes the bean definition.
	 *
	 * 设置 bean 的类加载器
	 *
	 * @param beanClassLoader the class loader to use,
	 * or {@code null} to suggest the default class loader
	 */
	void setBeanClassLoader(@Nullable ClassLoader beanClassLoader);

	/**
	 * Return this factory's class loader for loading bean classes
	 * (only {@code null} if even the system ClassLoader isn't accessible).
	 *
	 * 获取 bean 的类加载器
	 *
	 * @see org.springframework.util.ClassUtils#forName(String, ClassLoader)
	 */
	@Nullable
	ClassLoader getBeanClassLoader();

	/**
	 * Specify a temporary ClassLoader to use for type matching purposes.
	 * Default is none, simply using the standard bean ClassLoader.
	 * <p>A temporary ClassLoader is usually just specified if
	 * <i>load-time weaving</i> is involved, to make sure that actual bean
	 * classes are loaded as lazily as possible. The temporary loader is
	 * then removed once the BeanFactory completes its bootstrap phase.
	 *
	 * 设置临时加载器。如果涉及到加载时 aop 编织，通常仅指定一个临时类装入器，以确保实际的 bean 类被尽可能延迟地装入。
	 * 一旦 BeanFactory 完成他的启动解析后，这个临时的类加载器将被移除。
	 *
	 * @since 2.5
	 */
	void setTempClassLoader(@Nullable ClassLoader tempClassLoader);

	/**
	 * Return the temporary ClassLoader to use for type matching purposes,
	 * if any.
	 *
	 * 获取临时加载器
	 *
	 * @since 2.5
	 */
	@Nullable
	ClassLoader getTempClassLoader();
```



## bean 的元数据缓存

bean 的元数据缓存，默认为 true。如果为 false，每次创建 bean 都要从类加载器获取信息。

1. setCacheBeanMetadata 方法: 设置是否缓存。
2. isCacheBeanMetadata 方法: 判断是否缓存。

```java
	/**
	 * Set whether to cache bean metadata such as given bean definitions
	 * (in merged fashion) and resolved bean classes. Default is on.
	 * <p>Turn this flag off to enable hot-refreshing of bean definition objects
	 * and in particular bean classes. If this flag is off, any creation of a bean
	 * instance will re-query the bean class loader for newly resolved classes.
	 *
	 * bean 的元数据缓存，默认为 true。如果为 false，每次创建 bean 都要从类加载器获取信息。
	 * 设置是否缓存
	 */
	void setCacheBeanMetadata(boolean cacheBeanMetadata);

	/**
	 * Return whether to cache bean metadata such as given bean definitions
	 * (in merged fashion) and resolved bean classes.
	 *
	 * bean 的元数据缓存，默认为 true。如果为 false，每次创建 bean 都要从类加载器获取信息。
	 * 判断是否缓存
	 */
	boolean isCacheBeanMetadata();
```



## bean 的表达式解析器

1. setBeanExpressionResolver 方法: 设置表达式解析器。
2. getBeanExpressionResolver 方法: bean 的表达式解析器，获取表达式解析器。

```java
	/**
	 * Specify the resolution strategy for expressions in bean definition values.
	 * <p>There is no expression support active in a BeanFactory by default.
	 * An ApplicationContext will typically set a standard expression strategy
	 * here, supporting "#{...}" expressions in a Unified EL compatible style.
	 *
	 * bean 的表达式解析器，设置表达式解析器
	 *
	 * @since 3.0
	 */
	void setBeanExpressionResolver(@Nullable BeanExpressionResolver resolver);

	/**
	 * Return the resolution strategy for expressions in bean definition values.
	 *
	 * bean 的表达式解析器，获取表达式解析器
	 *
	 * @since 3.0
	 */
	@Nullable
	BeanExpressionResolver getBeanExpressionResolver();
```



## 类型转换器

1. setConversionService 方法: 设置类型转换器。
2. getConversionService 方法: 获取类型转换器。

```java
	/**
	 * Specify a Spring 3.0 ConversionService to use for converting
	 * property values, as an alternative to JavaBeans PropertyEditors.
	 *
	 * 设置类型转换器
	 *
	 * @since 3.0
	 */
	void setConversionService(@Nullable ConversionService conversionService);

	/**
	 * Return the associated ConversionService, if any.
	 *
	 * 获取类型转换器
	 *
	 * @since 3.0
	 */
	@Nullable
	ConversionService getConversionService();
```



## 属性编辑器

1. addPropertyEditorRegistrar 方法: 添加属性编辑器。
2. registerCustomEditor 方法: 注册给定类型的属性编辑器。

```java
	/**
	 * Add a PropertyEditorRegistrar to be applied to all bean creation processes.
	 * <p>Such a registrar creates new PropertyEditor instances and registers them
	 * on the given registry, fresh for each bean creation attempt. This avoids
	 * the need for synchronization on custom editors; hence, it is generally
	 * preferable to use this method instead of {@link #registerCustomEditor}.
	 *
	 * 添加属性编辑器
	 *
	 * @param registrar the PropertyEditorRegistrar to register
	 */
	void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);

	/**
	 * Register the given custom property editor for all properties of the
	 * given type. To be invoked during factory configuration.
	 * <p>Note that this method will register a shared custom editor instance;
	 * access to that instance will be synchronized for thread-safety. It is
	 * generally preferable to use {@link #addPropertyEditorRegistrar} instead
	 * of this method, to avoid for the need for synchronization on custom editors.
	 *
	 * 注册给定类型的属性编辑器
	 *
	 * @param requiredType type of the property
	 * @param propertyEditorClass the {@link PropertyEditor} class to register
	 */
	void registerCustomEditor(Class<?> requiredType, Class<? extends PropertyEditor> propertyEditorClass);
```


