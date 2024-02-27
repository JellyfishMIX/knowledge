# spring-ioc BeanDefinition 的解析与注册



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## BeanDefinition 来源

### xml 方式注册 BeanDefinition

#### bean 标签声明

```xml
<bean id="person" class="com.demo.springexplore.Person">
    <property name="name" value="xiaoming"/>
    <property name="age" value="18"/>
</bean>
```

### 注解方式注册 BeanDefinition

@Configuration + @Bean

```java
@Configuration
public class QuickstartConfiguration {
    @Bean
    public Person person() {
        return new Person();
    }
}
```

@Component + @ComponentScan

```java
@Configuration
@ComponentScan("com.demo.springexplore")
public class ComponentScanConfiguration {
}
```

#### @Import

可以传入四种类型: 普通类, 配置类, ImportSelector 的实现类，ImportBeanDefinitionRegistrar 的实现类

```java
public @interface Import {

	/**
	 * {@link Configuration @Configuration}, {@link ImportSelector},
	 * {@link ImportBeanDefinitionRegistrar}, or regular component classes to import.
	 */
	Class<?>[] value();

```

##### @Import 引入普通类

```java
@SpringBootApplication
// 通过 @Import 注解把 DemoBean 向 ApplicationContext 中注册 BeanDefinition
@Import(DemoBean.class)
public class MyBatisApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyBatisApplication.class, args);
    }
}
```

##### @Import 引入配置类

```java
@Configuration(proxyBeanMethods = false)
@Import({
        DemoDataConfiguration.PartOneConfiguration.class,
        DemoDataConfiguration.PartTwoConfiguration.class
})
public class DemoDataAutoConfiguration {
}

public class DemoDataConfiguration {

    @Configuration(proxyBeanMethods = false)
    static class PartOneConfiguration {
        @Bean
        @ConditionalOnMissingBean
        public BeanForIoc beanA() {
            return new BeanA();
        }
        
        @Bean
        @ConditionalOnMissingBean
        public BeanForIoc beanB() {
            return new BeanB();
        }
    }

    @Configuration(proxyBeanMethods = false)
    static class PartTwoConfiguration {
        // ....
    }
}
```

tips: 对于 springboot 项目，如果配置类在主启动类的包及子包中，是不需要 @Import 导入配置类的，springboot 帮忙做了。@Import 一般用于 @Configuration 标注的配置类不在主启动类同包下面，一般在自定义 starter 时用到。

##### @Import引入ImportSelector 的实现

```java
public interface ImportSelector {
    /**
     * 用于指定需要注册为 bean 的 Class 名称
     * 当在 @Configuration 标注的 Class 上使用 @Import 引入了一个 ImportSelector 实现类后，会把实现中返回的 Class 名称注册为 bean
     * 通过其参数 AnnotationMetadata importingClassMetadata 可以获取到 @Import 标注的 Class 的各种信息
     * 包括其 Class 名称，实现的接口名称, 父类名称, 添加的其它注解等信息，通过这些额外的信息可以辅助我们选择需要定义为 Spring bean 的 Class 名称
     */
    String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```

静态 import(import 固定的类)

```java
/**
 * XXXConfigurationSelector一定要配合@Import使用
 */
public class XXXConfigurationSelector implements ImportSelector {
    @Override
    @NonNull
    public String[] selectImports(@NonNull AnnotationMetadata importingClassMetadata) {
        // 返回需要注册为 bean 的 class 名称，外面会把 XXX 对应的类，向 BeanFactory 注册为 bean
        return new String[]{XXX.class.getName()};
    }
}

/**
 * 注意 @EnableXXX 注解真正起作用的是上面标注的 @Import(XXXConfigurationSelector.class)
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(XXXConfigurationSelector.class)
public @interface EnableXXX {
}

@SpringBootApplication
// 寓意使某个功能模块生效
@EnableXXX
public class MyBatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyBatisApplication.class, args);
    }

}
```

动态 import

1. 动态和静态 import 其实只是 selectImports 方法的实现内容不同。

2. 动态 import 获取 importingClassMetadata 的某些信息(例如注解)。

```java
public interface HelloService {
    void function();
}

public class DynamicSelectImport implements ImportSelector {
    /**
     * DynamicSelectImport 需要配合 @Import() 注解使用
     * <p>
     * 通过其参数 AnnotationMetadata importingClassMetadata 可以获取到 @Import 标注的 Class 的各种信息，
     * 包括其 Class 名称，实现的接口名称, 父类名称, 添加的其它注解等信息，通过这些额外的信息可以辅助我们选择需要注册为 bean 的 Class 名称
     * </p>
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 第一步，获取到通过 @ComponentScan 指定的包路径
        String[] basePackages = null;
        // @Import 注解对应的类上的 @ComponentScan 注解
        if (importingClassMetadata.hasAnnotation(ComponentScan.class.getName())) {
            Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(ComponentScan.class.getName());
            basePackages = (String[]) annotationAttributes.get("basePackages");
        }
        if (basePackages == null || basePackages.length == 0) {
            // ComponentScan 的 basePackages 默认为空数组
            String basePackage = null;
            try {
                // @Import 注解对应的类的包名
                basePackage = Class.forName(importingClassMetadata.getClassName()).getPackage().getName();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            basePackages = new String[]{basePackage};
        }
        // 第二步，获取指定包路径下所有实现了 HelloService 接口的类(ClassPathScanningCandidateComponentProvider 的使用)
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        Set<String> classes = new HashSet<>();
        for (String basePackage : basePackages) {
            scanner.findCandidateComponents(basePackage).forEach(beanDefinition -> classes.add(beanDefinition.getBeanClassName()));
        }
        // 把收集到的目标类名作为结果集返回
        return classes.toArray(new String[0]);
    }
}

@Configuration
// 指定包路径
@ComponentScan("com.tuacy.collect.mybatis")
@Import(DynamicSelectImport.class)
public class DynamicSelectConfig {
}
```

##### @Import 引入 ImportBeanDefinitionRegistrar 的实现

1. 关于 @Import 引入 ImportBeanDefinitionRegistrar 的案例，推荐看看 mybatis 关于 @MapperScan 的处理，很有意思。
2. 举一个非常简单的示例，直观感受下 ImportBeanDefinitionRegistrar 的使用。比如想把指定包路径下所有添加了自定义注解 @BeanIoc 的类注册为 bean，实现如下:

```java
/**
 * 标注了此注解的类会作为 bean 注册进 BeanFactory
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface BeanIoc {

}

/**
 * 定义包路径。(指定包下所有添加了 @BeanIoc 注解的类作为 bean)
 * 注意这里 @Import(BeanIocScannerRegister.class) 的使用
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(BeanIocScannerRegister.class)
public @interface BeanIocScan {
    String[] basePackages() default "";
}

/**
 * 搜索指定包下所有添加了 BeanIoc 注解的类，并且把这些类添加到 ioc 容器里面去
 */
public class BeanIocScannerRegister implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {
    private final static String PACKAGE_NAME_KEY = "basePackages";

    private ResourceLoader resourceLoader;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        // 1. 从 BeanIocScan 注解获取到我们要搜索的包路径
        AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(annotationMetadata.getAnnotationAttributes(BeanIocScan.class.getName()));
        if (annoAttrs == null || annoAttrs.isEmpty()) {
            return;
        }
        String[] basePackages = (String[]) annoAttrs.get(PACKAGE_NAME_KEY);
        // 2. 找到指定包路径下所有添加了 BeanIoc 注解的类，并且把这些类添加到 IOC 容器里面去
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(beanDefinitionRegistry, false);
        scanner.setResourceLoader(resourceLoader);
        scanner.addIncludeFilter(new AnnotationTypeFilter(BeanIoc.class));
        scanner.scan(basePackages);
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }
}

/**
 * 使用，使BeanIocScan生效
 */
@Configuration
@BeanIocScan(basePackages = "com.tuacy.collect.mybatis")
public class BeanIocScanConfig {
}
```

#### @Import 和 @ImportResource 的区别

@Import 用于导入类，@ImportResource 用于导入 xml 文件。

#### 模块化装配 @EnableXXX与 @Import 的结合

@EnableXXX 注解通常组合了 @Import 注解使用，达到批量引入一个功能模块的 bean 效果，减小心智负担。原理还是借助 @Import

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import({Red.class, ColorRegistrarConfiguration.class, ColorImportSelector.class, ColorImportBeanDefinitionRegistrar.class})
public @interface EnableColor {
    
}
```

### 编程方式注册 BeanDefinition

#### ImportBeanDefinitionRegistrar

```java
public class PersonRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
            .addPropertyValue("name", "xiaoming")
            .addPropertyValue("age", "18")
            .getBeanDefinition();
        registry.registerBeanDefinition("person", personDefinition);
    }
}
```

#### 手动向 AnnotationConfigApplicationContext 中注册 BeanDefinition

```java
public class AnnotationConfigApplication {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
            .addPropertyValue("name", "xiaoming")
            .addPropertyValue("age", "18")
            .getBeanDefinition();
        ctx.registerBeanDefinition("person", personDefinition);
        ctx.refresh();
    }
}
```

#### 实现自定义 BeanDefinitionRegistryPostProcessor

实现一个 BeanDefinitionRegistryPostProcessor，重写 postProcessBeanDefinitionRegistry 方法。

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
        .addPropertyValue("name", "xiaoming")
        .addPropertyValue("age", "18")
        .getBeanDefinition();
    registry.registerBeanDefinition("person", personDefinition);
}
```



## xml 和注解使用不同 ApplicationContext 实现

基于 xml 配置使用的实现是 ClassPathXmlApplicationContext，基于注解配置使用的实现是 AnnotationConfigApplicationContext

基于注解配置的 AnnotationConfigApplicationContext 示例

```java
public class AnnotationConfigApplication {
    public static void main(String[] args) throws Exception {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(QuickstartConfiguration.class);
        Person person = ctx.getBean(Person.class);
        System.out.println(person);
    }
}

@Configuration
public class QuickstartConfiguration {
    @Bean
    public Person person() {
        return new Person();
    }
}
```

### xml 配置与注解配置引入另一方

ClassPathXmlApplicationContext 和 AnnotationConfigApplicationContext 均无法即基于 xml 又基于注解，只能基于单一的一种方式。

因此如果两种配置方式都想使用，要确定一种主要的配置方式来选择使用的 ApplicationContext 的实现，然后引入另一种。

#### 基于 xml 引入注解

通过 context:annotation-config 可以在 xml 中开启注解配置类，然后配置一个注解配置类 bean 即可。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd 
        http://www.springframework.org/schema/context 
        https://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 开启注解配置 -->
    <context:annotation-config/>
    <bean class="com.demo.springexplore.QuickstartConfiguration"/>
</beans>
```

#### 基于注解引入 xml

在注解配置类中使用 @ImportResource 可引入 xml 配置文件

```java
@Configuration
@ImportResource("classpath:spring/bean.xml")
public class ImportXmlAnnotationConfiguration {
    
}
```



## BeanDefinition 的注册流程

### 注册时机

1. BeanDefinition 的注册时机最早是 AbstractApplicationContext#refresh 刷新上下文中的 obtainFreshBeanFactory 步骤。

```java
	/**
	 * 刷新 spring 应用的上下文
	 * 此方法大量使用了模版方法模式，规定了刷新应用上下文统一的动作，具体动作实现逻辑可供子类定制化。
	 */
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		// 获取启动关闭锁，线程安全地刷新 spring 应用上下文
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。
			prepareRefresh();

			/*
			 * Tell the subclass to refresh the internal bean factory.
			 *
			 * 初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，可以进行 bean 的获取等基础操作了。
			 */
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 对 beanFactory 做了一些准备工作，进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 留给子类扩展，可以对 BeanFactory 做后置处理
				postProcessBeanFactory(beanFactory);

				/*
				 * Invoke factory processors registered as beans in the context.
				 *
				 * 调用各种 BeanFactoryPostProcessor
				 * 其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。
				 */
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册 BeanPostProcessor
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 为上下文初始化 MessageSource，可以对不同语言的消息体进行国际化处理
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化应用事件广播器
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 留给子类的扩展方法，是一个 Context 的生命周期函数。例如子类可以用来初始化其他 bean
				onRefresh();

				// Check for listener beans and register them.
				// 把 ApplicationListener 的 bean 实例添加进应用事件广播器
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 非延迟地初始化剩下的实例，这里实例化 bean 调用了 ConfigurableListableBeanFactory#getBean 方法
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 完成刷新过程，调用生命周期处理器 LifecycleProcessor#onRefresh 方法。并且发布 ContextRefreshEvent。
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```



## xml 方式注册 BeanDefinition 原理

### 时机 -- obtainFreshBeanFactory

1. 创建 BeanFactory。
2. 加载配置文件。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

protected final void refreshBeanFactory() throws BeansException {
    // 存在 BeanFactory 则先销毁
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 创建 BeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        beanFactory.setSerializationId(getId());
        customizeBeanFactory(beanFactory);
        // 加载 BeanDefinition
        loadBeanDefinitions(beanFactory);
        this.beanFactory = beanFactory;
    } // catch ......
}
```



```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // xml 配置文件由 XmlBeanDefinitionReader 解析
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // 配置上下文环境, 资源加载器等
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    initBeanDefinitionReader(beanDefinitionReader);
    // 使用 xml 解析器, 解析 xml 配置文件
    loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        // 加载配置文件资源路径的 xml 配置文件
        reader.loadBeanDefinitions(configLocations);
    }
}
```

### XmlBeanDefinitionReader 加载 BeanDefinition

1. 传入一组 xml 配置文件路径表达式，遍历加载每一个表达式。
2. 具体的解析逻辑比较繁琐暂不关注。

```java
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    // assert ......
    int count = 0;
    for (String location : locations) {
        count += loadBeanDefinitions(location);
    }
    return count;
}
```



## 注解方式注册 BeanDefinition 原理

### 时机 -- invokeBeanFactoryPostProcessors

1. 在 AbstractApplicationContext#refresh 时调用 invokeBeanFactoryPostProcessors 方法，使用 BeanDefinitionRegistryPostProcessor 责任链式处理 BeanDefinitionRegistry 时，其中一个实现 ConfigurationClassPostProcessor 中。

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		// 获取启动关闭锁，线程安全地刷新 spring 应用上下文
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 为刷新上下文作准备，例如对系统属性或者环境变量进行准备及验证。
			prepareRefresh();

			/*
			 * Tell the subclass to refresh the internal bean factory.
			 *
			 * 初始化 beanFactory，之后 ApplicationContext 就具有了 BeanFactory 所提供的功能，可以进行 bean 的获取等基础操作了。
			 */
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 对 beanFactory 做了一些准备工作，进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 留给子类扩展，可以对 BeanFactory 做后置处理
				postProcessBeanFactory(beanFactory);

				/*
				 * Invoke factory processors registered as beans in the context.
				 *
				 * 调用各种 BeanFactoryPostProcessor
				 * 其中最关键的是 ConfigurationClassPostProcessor，在这里完成了配置类的解析，生成配置类的 BeanDefinition。
				 */
				invokeBeanFactoryPostProcessors(beanFactory);
                
                // ...
            }
        }
    }       
```



1. 将 BeanDefinitionRegistryPostProcessor 与 BeanFactoryPostProcessor 分离开。
2. 按照排序-执行-清理的操作流程，先操作实现了 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessors，再操作实现了 Ordered 接口的 BeanDefinitionRegistryPostProcessors，最后操作其他的 BeanDefinitionRegistryPostProcessor。
3. 执行 BeanFactoryPostProcessor。

```java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				// 将 BeanDefinitionRegistryPostProcessor 与 BeanFactoryPostProcessor 分离开
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			// 先执行实现了 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessors
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
            // 排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			// 具体地执行 BeanDefinitionRegistryPostProcessor
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            // 清理
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			// 再执行实现了 Ordered 接口的 BeanDefinitionRegistryPostProcessors
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
            // 排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			// 具体地执行 BeanDefinitionRegistryPostProcessor
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            // 清理
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			// 最后执行其他的 BeanDefinitionRegistryPostProcessor
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
                // 排序
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				// 具体地执行 BeanDefinitionRegistryPostProcessor
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                // 清理
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// 执行 BeanFactoryPostProcessor
    	// ...
	}
```

### 具体地执行 BeanDefinitionRegistryPostProcessor

1. 责任链模式调用 BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry

```java
	/**
	 * 具体地执行 BeanDefinitionRegistryPostProcessor
	 */
	private static void invokeBeanDefinitionRegistryPostProcessors(
			Collection<? extends BeanDefinitionRegistryPostProcessor> postProcessors, BeanDefinitionRegistry registry) {

        // 责任链模式调用 BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
		for (BeanDefinitionRegistryPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessBeanDefinitionRegistry(registry);
		}
	}
```

### 解析注解方式配置的 BeanDefinition 的关键实现 -- ConfigurationClassPostProcessor

org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions

1. 筛选出所有的配置 BeanDefinition，后面仅解析并注册筛选出的结果集。
   1. 这里可以看出，通过 xml 配置的 bean 是不在这里解析的(上文已介绍 xml 的解析位置)。这里只解析 @Component, @ComponentScan, @Import, @ImportResource, @Bean 及它们的派生注解声明的 BeanDefinition。后面会说明。
2. 构造默认的 BeanNameGenerator(beanName 生成器)
3. 解析配置 bean，加载配置 bean

```java
	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		// 筛选出所有的配置 BeanDefinition，后面仅解析筛选出的结果集
		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}

		// Sort by previously determined @Order value, if applicable
		// 配置 BeanDefinition 排序
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

		// Detect any custom bean name generation strategy supplied through the enclosing application context
		// 构造默认的 BeanNameGenerator(beanName 生成器)
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
						AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					this.componentScanBeanNameGenerator = generator;
					this.importBeanNameGenerator = generator;
				}
			}
		}

		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}

		// Parse each @Configuration class
		// 真正解析配置 BeanDefinition 的组件: ConfigurationClassParser
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
			// 解析配置 BeanDefinition
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			// 加载配置 bean 的 BeanDefinition
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}
				for (String candidateName : newCandidateNames) {
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		while (!candidates.isEmpty());

		// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
		if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
			sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
		}

		if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
			// Clear cache in externally provided MetadataReaderFactory; this is a no-op
			// for a shared cache since it'll be cleared by the ApplicationContext.
			((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
		}
	}
```

#### 判断是否为配置 BeanDefinition

##### 过渡方法

```java
// org.springframework.context.annotation.ConfigurationClassUtils#checkConfigurationClassCandidate

public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

		String className = beanDef.getBeanClassName();
		if (className == null || beanDef.getFactoryMethodName() != null) {
			return false;
		}

		AnnotationMetadata metadata;
		if (beanDef instanceof AnnotatedBeanDefinition &&
				className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
			// Can reuse the pre-parsed metadata from the given BeanDefinition...
			metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
		}
		else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
			// Check already loaded Class if present...
			// since we possibly can't even load the class file for this Class.
			Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
			if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
					BeanPostProcessor.class.isAssignableFrom(beanClass) ||
					AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
					EventListenerFactory.class.isAssignableFrom(beanClass)) {
				return false;
			}
			metadata = AnnotationMetadata.introspect(beanClass);
		}
		else {
			try {
				MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
				metadata = metadataReader.getAnnotationMetadata();
			}
			catch (IOException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Could not find class file for introspecting configuration annotations: " +
							className, ex);
				}
				return false;
			}
		}

		Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
		// 关键，判断是否为配置 BeanDefinition
		else if (config != null || isConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
		else {
			return false;
		}

		// It's a full or lite configuration candidate... Let's determine the order value, if any.
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
```

##### 关键，判断是否为配置 BeanDefinition

1. 判断当前 bean 是否为 @Component, @ComponentScan, @Import, @ImportResource, @Bean 及它们的派生注解声明的 BeanDefinition
2. 第一个当前 bean 来源: 
   1. 如果使用基于 xml 的 ClassPathXmlApplicationContext，为了引入注解式配置在 xml 中配置的配置类，就是第一个(或前几个)当前 bean 的来源。
   2. 如果使用基于注解的 AnnotationConfigApplicationContext，一般会通过 AnnotationConfigApplicationContext 的构造方法/builder 传入第一个或前几个配置类。例如 springboot 在构建 AnnotationConfigApplicationContext 时就把主启动类传进去了，主启动类上的 @SpringBootApplication 注解带有 @ComponentScan 注解，默认扫描和主启动类同包下的路径。

```java
	static {
		candidateIndicators.add(Component.class.getName());
		candidateIndicators.add(ComponentScan.class.getName());
		candidateIndicators.add(Import.class.getName());
		candidateIndicators.add(ImportResource.class.getName());
	}

	public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {
		// Do not consider an interface or an annotation...
		if (metadata.isInterface()) {
			return false;
		}

		// Any of the typical annotations found?
		for (String indicator : candidateIndicators) {
			if (metadata.isAnnotated(indicator)) {
				return true;
			}
		}

		// Finally, let's look for @Bean methods...
		return hasBeanMethods(metadata);
	}

	static boolean hasBeanMethods(AnnotationMetadata metadata) {
		try {
			return metadata.hasAnnotatedMethods(Bean.class.getName());
		}
		catch (Throwable ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
			}
			return false;
		}
	}
```

#### 解析配置 BeanDefinition

##### 过渡方法

1. 解析主要逻辑。
2. 处理 DeferredImportSelector 接口的实现。

```java
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				if (bd instanceof AnnotatedBeanDefinition) {
					parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
					parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}

        // 处理 DeferredImportSelector 接口的实现
		this.deferredImportSelectorHandler.process();
	}

	protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
		processConfigurationClass(new ConfigurationClass(metadata, beanName), DEFAULT_EXCLUSION_FILTER);
	}

	protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}

		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		if (existingClass != null) {
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				return;
			}
			else {
				// Explicit bean definition found, probably replacing an import.
				// Let's remove the old one and go with the new one.
				this.configurationClasses.remove(configClass);
				this.knownSuperclasses.values().removeIf(configClass::equals);
			}
		}

		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = asSourceClass(configClass, filter);
		do {
			sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
		}
		while (sourceClass != null);

		this.configurationClasses.put(configClass, configClass);
	}
```

##### 解析各种注解的关键方法

1. 可以看到对 @PropertySources, @ComponentScan, @Import, @ImportResource, @Bean 等注解分别进行了处理。

```java
	@Nullable
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {

		if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
			// Recursively process any member (nested) classes first
			processMemberClasses(configClass, sourceClass, filter);
		}

		// Process any @PropertySource annotations
		// 处理 @PropertySources
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			}
			else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}

		// Process any @ComponentScan annotations
		// 处理 @ComponentScan
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

		// Process any @Import annotations
        // 处理 @Import 注解
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

		// Process any @ImportResource annotations
		// 处理 @ImportResource
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}

		// Process individual @Bean methods
		// 处理 @Bean 标注的方法
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}

		// Process default methods on interfaces
		processInterfaces(configClass, sourceClass);

		// Process superclass, if any
		if (sourceClass.getMetadata().hasSuperClass()) {
			String superclass = sourceClass.getMetadata().getSuperClassName();
			if (superclass != null && !superclass.startsWith("java") &&
					!this.knownSuperclasses.containsKey(superclass)) {
				this.knownSuperclasses.put(superclass, configClass);
				// Superclass found, return its annotation metadata and recurse
				return sourceClass.getSuperClass();
			}
		}

		// No superclass -> processing is complete
		return null;
	}
```

##### 处理 DeferredImportSelector 接口的实现

在 springframkework 4.0 中，ImportSelector 多了一个子接口: DeferredImportSelector，它的执行时机比 ImportSelector 更晚，它会在注解声明的配置类的所有解析工作完成后才执行(上文的源码就已经解释了这个原理)。

一般情况下，DeferredImportSelector 会跟 @Conditional 条件装配注解配合使用。此外 springboot 自动装配也使用了 DeferredImportSelector 实现之一 AutoConfigurationImportSelector。

```java
// org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorHandler#process

public void process() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    this.deferredImportSelectors = null;
    try {
        if (deferredImports != null) {
            DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
            deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
            // 遍历所有 DeferredImportSelector 的实现，依次解析
            deferredImports.forEach(handler::register);
            handler.processGroupImports();
        }
    }
    finally {
        this.deferredImportSelectors = new ArrayList<>();
    }
}
```



## 加载配置 BeanDefinition

1. 遍历所有配置类迭代加载
2. 如果当前配置类是被@Import的，把自己注册进 BeanFactory
3. 从 @ImportResource 中指定的文件加载配置 BeanDefinition
4. 从 ImportBeanDefinitionRegistrar 中加载配置 BeanDefinition

```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    // 遍历所有配置类迭代加载
    for (ConfigurationClass configClass : configurationModel) {
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}

	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

		// 如果条件装配将其跳过，则对应的 BeanDefinition 不会注册进 BeanDefinitionRegistry
		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}

		// 如果当前配置类是被@Import的，把自己注册进 BeanFactory
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			// 加载 @Bean 注解声明的 BeanDefinition
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}

		// 从 @ImportResource 中指定的文件加载配置 BeanDefinition
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		// 从 ImportBeanDefinitionRegistrar 中加载配置 BeanDefinition
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```

### ImportBeanDefinitionRegistrar

ImportBeanDefinitionRegistrar 需要由 @Import 导入，若未被其他 bean 通过 @Import 导入则无法触发加载。



## 编程方式注册 BeanDefinition 原理

### ImportBeanDefinitionRegistrar

实现一个 ImportBeanDefinitionRegistrar，重写 registerBeanDefinitions 方法，在其他 bean 中通过 @Import 导入此实现，则会触发 ImportBeanDefinitionRegistrar 实现中 BeanDefinition 的加载。

```java
public class PersonRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
            .addPropertyValue("name", "xiaoming")
            .addPropertyValue("age", "18")
            .getBeanDefinition();
        registry.registerBeanDefinition("person", personDefinition);
    }
}
```

### 手动向 AnnotationConfigApplicationContext 中注册 BeanDefinition

1. 这种原理没什么可说的，就是手动向 ApplicationContext 注册一个 BeanDefinition。

```java
public class AnnotationConfigApplication {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
        BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
            .addPropertyValue("name", "xiaoming")
            .addPropertyValue("age", "18")
            .getBeanDefinition();
        ctx.registerBeanDefinition("person", personDefinition);
        ctx.refresh();
    }
}
```

### 实现自定义 BeanDefinitionRegistryPostProcessor

1. 原理是实现一个 BeanDefinitionRegistryPostProcessor，重写 postProcessBeanDefinitionRegistry 方法，向 BeanDefinition 注册一个 BeanDefinition，就像 ConfigurationClassPostProcessor 所做的那样。

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    BeanDefinition personDefinition = BeanDefinitionBuilder.rootBeanDefinition(Person.class)
        .addPropertyValue("name", "xiaoming")
        .addPropertyValue("age", "18")
        .getBeanDefinition();
    registry.registerBeanDefinition("person", personDefinition);
}
```

