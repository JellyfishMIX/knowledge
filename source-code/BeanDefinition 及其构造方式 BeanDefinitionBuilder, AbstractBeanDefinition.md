# BeanDefinition 及其构造方式 BeanDefinitionBuilder, AbstractBeanDefinition



## BeanDefinition

### BeanDefinition 的属性

spring IoC 容器中的每一个 bean 都会有一个对应的 BeanDefinition 实例，该实例负责保存 bean 对象的所有必要信息，如下所示：

| 属性(property)           | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| class                    | bean 全类名，必须是具体类，不能是抽象类或接口(因为抽象类或接口不能实例化) |
| name                     | bean 的名称或 id                                             |
| scope                    | bean 的作用域                                                |
| constructor arguments    | bean 构造器参数(用于依赖注入)                                |
| properties               | bean 属性设置(用于依赖注入)                                  |
| autowiring mode          | bean 自动绑定模式(例如通过名称 byName)                       |
| lazy initialization mode | bean 延迟初始化模式(延迟和非延迟)                            |
| initialization method    | bean 初始化回调方法名称                                      |
| destruction method       | bean 销毁回调方法名称                                        |



> 需要说明的一点是：如果是自己直接通过 `SingletonBeanRegistry#registerSingleton` 向容器手动注入 Bean 的，不会存在这份 Bean 定义信息，这点需要注意。 Spring 内部有不少这样的例子（因为这种Bean非常简单，不需要定义信息）： 
>
> ```java
> beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
> bf.registerSingleton(WebApplicationContext.SERVLET_CONTEXT_BEAN_NAME, servletContext);
> Collections.unmodifiableMap(attributeMap));
> ```

### 三种 BeanDefinition:

1. GenericBeanDefinition: 通用的 BeanDefinition，可以有 parentBeanDefinition。源码的实现非常简单，只增加了一个 parentName 的属性值，其余的实现都在父类 AbstractBeanDefinition 里。
   1. 通过 xml 方式配置 bean，最初被加载进来都是一个 GenericBeanDefinition，之后再逐渐解析的。
2. ChildBeanDefinition: 子 BeanDefinition，依赖于父类 RootBeanDefinition
   1. 它可以继承它父类的设置，即 ChildBeanDefinition 对 RootBeanDefinition 有一定的依赖关系。
   2. 从 spring 2.5 开始，提供了一个更好的 GenericBeanDefinition，所以以后推荐使用它，不使用 ChildBeanDefinition。
3. RootBeanDefinition: 根 BeanDefinition，不能有 parentBeanDefinition
   1. 在 配置文件中可以定义父和子，子用 ChildBeanDefiniton 表示，而没有父的用 RootBeanDefinition 表示。

#### BeanDefinitionBuilder 中与三种 BeanDefinition 对应的构造方法

```java
/**
 * Programmatic means of constructing
 * {@link org.springframework.beans.factory.config.BeanDefinition BeanDefinitions}
 * using the builder pattern. Intended primarily for use when implementing Spring 2.0
 * {@link org.springframework.beans.factory.xml.NamespaceHandler NamespaceHandlers}.
 *
 * @author Rod Johnson
 * @author Rob Harrop
 * @author Juergen Hoeller
 * @since 2.0
 */
public final class BeanDefinitionBuilder {

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link GenericBeanDefinition}.
	 *
	 * 创建一个通用的 beanDefinition，什么属性都不填充
	 */
	public static BeanDefinitionBuilder genericBeanDefinition() {
		return new BeanDefinitionBuilder(new GenericBeanDefinition());
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link GenericBeanDefinition}.
	 *
	 * 创建一个通用的 beanDefinition，填充 bean 的 className
	 *
	 * @param beanClassName the class name for the bean that the definition is being created for
	 */
	public static BeanDefinitionBuilder genericBeanDefinition(String beanClassName) {
		BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new GenericBeanDefinition());
		builder.beanDefinition.setBeanClassName(beanClassName);
		return builder;
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link GenericBeanDefinition}.
	 *
	 * 创建一个通用的 beanDefinition，填充 bean 的 class
	 *
	 * @param beanClass the {@code Class} of the bean that the definition is being created for
	 */
	public static BeanDefinitionBuilder genericBeanDefinition(Class<?> beanClass) {
		BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new GenericBeanDefinition());
		builder.beanDefinition.setBeanClass(beanClass);
		return builder;
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link GenericBeanDefinition}.
	 *
	 * 创建一个通用的 beanDefinition，填充 bean 的 class，填充一个 bean 实例创建时会调用的回调函数
	 *
	 * @param beanClass the {@code Class} of the bean that the definition is being created for
	 * @param instanceSupplier a callback for creating an instance of the bean
	 * @since 5.0
	 */
	public static <T> BeanDefinitionBuilder genericBeanDefinition(Class<T> beanClass, Supplier<T> instanceSupplier) {
		BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new GenericBeanDefinition());
		builder.beanDefinition.setBeanClass(beanClass);
		builder.beanDefinition.setInstanceSupplier(instanceSupplier);
		return builder;
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link RootBeanDefinition}.
	 *
	 * 创建一个根 beanDefinition，填充 bean 的 className
	 *
	 * @param beanClassName the class name for the bean that the definition is being created for
	 */
	public static BeanDefinitionBuilder rootBeanDefinition(String beanClassName) {
		return rootBeanDefinition(beanClassName, null);
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link RootBeanDefinition}.
	 *
	 * 创建一个根 beanDefinition，填充 bean 的 className，填充构造 bean 实例使用的方法名
	 *
	 * @param beanClassName the class name for the bean that the definition is being created for
	 * @param factoryMethodName the name of the method to use to construct the bean instance  构造 bean 实例使用的方法名
	 */
	public static BeanDefinitionBuilder rootBeanDefinition(String beanClassName, @Nullable String factoryMethodName) {
		BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new RootBeanDefinition());
		builder.beanDefinition.setBeanClassName(beanClassName);
		builder.beanDefinition.setFactoryMethodName(factoryMethodName);
		return builder;
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link RootBeanDefinition}.
	 *
	 * 创建一个根 beanDefinition，填充 bean 的 class
	 *
	 * @param beanClass the {@code Class} of the bean that the definition is being created for
	 */
	public static BeanDefinitionBuilder rootBeanDefinition(Class<?> beanClass) {
		return rootBeanDefinition(beanClass, null);
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link RootBeanDefinition}.
	 *
	 * 创建一个根 beanDefinition，填充 bean 的 class，填充构造 bean 实例使用的方法名
	 *
	 * @param beanClass the {@code Class} of the bean that the definition is being created for
	 * @param factoryMethodName the name of the method to use to construct the bean instance
	 */
	public static BeanDefinitionBuilder rootBeanDefinition(Class<?> beanClass, @Nullable String factoryMethodName) {
		BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new RootBeanDefinition());
		builder.beanDefinition.setBeanClass(beanClass);
		builder.beanDefinition.setFactoryMethodName(factoryMethodName);
		return builder;
	}

	/**
	 * Create a new {@code BeanDefinitionBuilder} used to construct a {@link ChildBeanDefinition}.
	 *
	 * 创建一个子 beanDefinition，填充父 beanDefinition 的名字
	 *
	 * @param parentName the name of the parent bean
	 */
	public static BeanDefinitionBuilder childBeanDefinition(String parentName) {
		return new BeanDefinitionBuilder(new ChildBeanDefinition(parentName));
	}


	/**
	 * The {@code BeanDefinition} instance we are creating.
	 *
	 * 创建出的 beanDefinition 实例
	 */
	private final AbstractBeanDefinition beanDefinition;
```



## AbstractBeanDefinition

BeanDefinition 是个接口，AbstractBeanDefinition 是这个接口的实现类，很多常见的 bean 属性在 AbstractBeanDefinition 中。这是经典的工厂模式，抽象出接口去规范工厂生产的实体类的行为。

另外 GenericBeanDefinition 只是子类实现，大部分的属性在 AbstractBeanDefinition 中。

```java
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
        implements BeanDefinition, Cloneable {
    /**
	 * bean 对应的 class（用一个 Class 对象来表示）
	 */
	@Nullable
	private volatile Object beanClass;

	/**
	 * bean 的作用范围
	 * 对应 bean 属性 scope
	 */
	@Nullable
	private String scope = SCOPE_DEFAULT;

	/**
	 * 是否是抽象类
	 * 对应 bean 属性 abstract
	 */
	private boolean abstractFlag = false;

	/**
	 * 是否延迟加载
	 * 对应 bean 属性 lazy-init
	 */
	@Nullable
	private Boolean lazyInit;

	/**
	 * 自动装配模式，自动注入模式
	 * 对应 bean 属性 autowire
	 */
	private int autowireMode = AUTOWIRE_NO;

	/**
	 * 依赖检查模式
	 */
	private int dependencyCheck = DEPENDENCY_CHECK_NONE;

	/**
	 * 用来表示一个 bean 的实例化依靠另一个 bean 先实例化
	 * 这里只会存放<bean/>标签的depends-on属性或是@DependsOn注解的值
	 */
	@Nullable
	private String[] dependsOn;

	/**
	 * autowire-candidate 属性设置为 false，这样容器在查找自动装配对象时，
	 * 将不考虑该 bean，即它不会被考虑作为其他 bean 自动装配的候选者，
	 * 但是该 bean 本身还是可以使用自动装配来注入其他 bean 的
	 */
	private boolean autowireCandidate = true;

	/**
	 * 当某个 bean 的某个属性自动装配有多个候选者（候选者包括此 bean）时，是否优先注入，即 @Primary 注解
	 * 对应 bean 属性 primary
	 */
	private boolean primary = false;

	/**
	 * 用于记录 qualifier
	 * AutowireCandidateQualifier 用于解析自动装配的候选者
	 */
	private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

	/**
	 * 用于初始化 bean 的回调函数，一旦指定，这个方法会覆盖工厂方法以及构造函数中的元数据
	 * 可以理解为，通过这个函数的逻辑初始化 bean，而不是构造函数或是工厂方法
	 */
	@Nullable
	private Supplier<?> instanceSupplier;

	/**
	 * 是否允许访问非 public 方法和属性，应用场景是构造函数, 工厂方法, init, destroy 方法解析构造函数
	 */
	private boolean nonPublicAccessAllowed = true;

	/**
	 * 指定解析构造函数的模式，是宽松还是严格
	 * 如果为false，则在以下情况
	 * interface ITest{}
	 * class ITestImpl implements ITest{};
	 * class Main {
	 *     Main(ITest i) {}
	 *     Main(ITestImpl i) {}
	 * }
	 * 抛出异常，因为Spring无法准确定位哪个构造函数程序设置
	 */
	private boolean lenientConstructorResolution = true;

	/**
	 * bean 的 factoryBeanName
	 * <bean id = "currentTime" factory-bean = "instanceFactoryBean" factory-method = "createTime" />
	 */
	@Nullable
	private String factoryBeanName;

	/**
	 * bean 的 factoryMethodName
	 * <bean id = "currentTime" factory-bean = "instanceFactoryBean" factory-method = "createTime" />
	 */
	@Nullable
	private String factoryMethodName;

	/**
	 * 记录构造函数注入属性，对应 bean 属性 constructor-arg
	 */
	@Nullable
	private ConstructorArgumentValues constructorArgumentValues;

	/**
	 * 普通属性集合
	 */
	@Nullable
	private MutablePropertyValues propertyValues;

	/**
	 * 方法重写的持有者，记录 lookup-method, replaced-method 元素
	 */
	private MethodOverrides methodOverrides = new MethodOverrides();

	/**
	 * 初始化方法
	 */
	@Nullable
	private String initMethodName;

	/**
	 * 销毁方法
	 */
	@Nullable
	private String destroyMethodName;

	/**
	 * 是否执行 init-method
	 */
	private boolean enforceInitMethod = true;

	/**
	 * 是否执行 destroy-method
	 */
	private boolean enforceDestroyMethod = true;

	/**
	 * 是否是合成类。根据用户自定义的类生成的 bean 不是合成类，应用程序(框架)生成的是合成类(例如生成 aop 代理类时，不是用户自定义的，这个就是合成类)
	 */
	private boolean synthetic = false;

	/**
	 * 这个 bean 的角色，ROLE_APPLICATION：用户，ROLE_INFRASTRUCTURE：完全内部使用，与用户无关，
	 * ROLE_SUPPORT：某些庞大配置的一部分
	 */
	private int role = BeanDefinition.ROLE_APPLICATION;

	/**
	 * bean 的描述
	 */
	@Nullable
	private String description;

	/**
	 * 这个 bean 定义的资源
	 */
	@Nullable
	private Resource resource;
}
```

AbstractBeanDefinition 定义了一系列描述 bean 的属性，可以看到 bean 属性的某些默认值（例如默认为单例）。这个类的属性部分可以在 xml 配置文件中找到相应的属性或是标签（例如 AbstractBeanDefinition 的 scope 属性对应 xml 配置文件中的 scope 属性）。



## BeanDefinition 的构造方式

1. 通过 BeanDefinitionBuilder
2. 通过 AbstractBeanDefinition 及其派生类

```java
    public static void main(String[] args) {
        // 1. 通过 BeanDefinitionBuilder 构建
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(User.class);
        // 设置 bean 属性，builder 模式可以做到链式调用
        beanDefinitionBuilder.addPropertyValue("id", 1).addPropertyValue("name", "小明");
        // 获取 BeanDefinition 实例
        BeanDefinition abstractBeanDefinition = beanDefinitionBuilder.getBeanDefinition();

        // BeanDefinition 不是 Bean 的终态，可以自定义修改

        // 2. 通过 AbstractBeanDefinition 及其派生类构建
        AbstractBeanDefinition genericBeanDefinition = new GenericBeanDefinition();
        // 设置 bean 类型
        genericBeanDefinition.setBeanClass(User.class);
        // 通过 MutablePropertyValues 批量操作属性，这也是一种链式调用
        MutablePropertyValues propertyValues = new MutablePropertyValues();
        // propertyValues.add("id", 1);
        // propertyValues.add("name", "小明");
        propertyValues.add("id", 1).add("name", "小明");
        genericBeanDefinition.setPropertyValues(propertyValues);
    }
```



## BeanDefinition 什么时候会用到？

spirng 的 ioc 容器实例化 bean 的过程需要 BeanDefinition 中的信息，它是实例化 Bean 的原材料。

例如 org.springframework.context.config.AbstractPropertyLoadingBeanDefinitionParser，这是一个 BeanDefinitionParser (接口)的实现类，用来解析 xml 中的 context:property-... 这样的元素。

### 解析 xml 时使用 BeanDefinition

```java
package org.springframework.beans.factory.xml;

/**
 * Abstract parser for &lt;context:property-.../&gt; elements.
 * 
 *
 *
 * @author Juergen Hoeller
 * @author Arjen Poutsma
 * @author Dave Syer
 * @since 2.5.2
 */
abstract class AbstractPropertyLoadingBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {	
    
	@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		String location = element.getAttribute("location");
		if (StringUtils.hasLength(location)) {
			location = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(location);
			String[] locations = StringUtils.commaDelimitedListToStringArray(location);
			builder.addPropertyValue("locations", locations);
		}

		String propertiesRef = element.getAttribute("properties-ref");
		if (StringUtils.hasLength(propertiesRef)) {
			builder.addPropertyReference("properties", propertiesRef);
		}

		String fileEncoding = element.getAttribute("file-encoding");
		if (StringUtils.hasLength(fileEncoding)) {
			builder.addPropertyValue("fileEncoding", fileEncoding);
		}

		String order = element.getAttribute("order");
		if (StringUtils.hasLength(order)) {
			builder.addPropertyValue("order", Integer.valueOf(order));
		}

		builder.addPropertyValue("ignoreResourceNotFound",
				Boolean.valueOf(element.getAttribute("ignore-resource-not-found")));

		builder.addPropertyValue("localOverride",
				Boolean.valueOf(element.getAttribute("local-override")));

		builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	}
    
}
```

具体的逻辑我们可以先不管，能看到 builder.addPropertyValue 这样的代码，这里的 doParse 方法一边在解析 xml，一边在使用 BeanDefinitionBuilder 构建 BeanDefinition。

### 通过 BeanDefinition 实例化 bean

在一个 AbstractBeanFactory 的子类中，可以看到有一个方法 doCreateBean 在根据 BeanDefinition 实例化 bean。

```java
package org.springframework.beans.factory.support;

public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
		implements AutowireCapableBeanFactory {
	/**
	 * Actually create the specified bean. Pre-creation processing has already happened
	 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
	 * <p>Differentiates between default bean instantiation, use of a
	 * factory method, and autowiring a constructor.
	 * @param beanName the name of the bean
	 * @param mbd the merged bean definition for the bean
	 * @param args explicit arguments to use for constructor or factory method invocation
	 * @return a new instance of the bean
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
```

bean 构建的具体逻辑我们以后再分析。这里只是举例一下 BeanDefinition 用到的地方，对 BeanDefinition 从构建到使用有一个直观的印象。