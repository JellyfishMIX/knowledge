# springboot 运行 SpringApplication -- SpringApplication#run



## 说明

1. 本文基于 springboot 2.1.x 编写
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## SpringApplication#run 方法

org.springframework.boot.SpringApplication#run(java.lang.String...)

```java
public ConfigurableApplicationContext run(String... args) {
    // 创建 StopWatch 对象
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    // 声明 ioc 容器和一组 ExceptionReporter 集合
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 配置与 awt 相关的信息
    configureHeadlessProperty();
    // 获取 SpringApplicationRunListeners，并调用 starting 方法(回调机制)
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 回调 SpringApplicationRunListener，时机是 SpringApplication#run 方法执行时立即调用。
    listeners.starting();
    try {
        // 将 main 方法的 args 参数封装到一个对象中
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备 Environment
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 如果有配置 spring.beaninfo.ignore，则将该配置设置进系统参数
        configureIgnoreBeanInfo(environment);
        // 打印 springboot 的 banner
        Banner printedBanner = printBanner(environment);
        // 创建 applicationContext
        context = createApplicationContext();
        // 初始化异常报告器
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // 准备 ApplicationContext
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 触发 ApplicationContext refresh
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // ApplicationContext refresh 后，application 已启动，但尚未调用 CommandLineRunners 和 ApplicationRunners。
        listeners.started(context);
        callRunners(context, applicationArguments);
    } catch(Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }
    
    try {
        listeners.running(context);
    } catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

 

## StopWatch

无需关注，验证性功能。

> This class is normally used to verify performance during proof-of-concepts and in development, rather than as part of production applications.
>
> 常用于在概念验证和开发过程中验证性能，而不是作为生产应用程序的一部分。



## SpringBootExceptionReporter

不重要，暂不关注。



## configureHeadlessProperty

不重要，暂不关注。



## SpringApplicationRunListeners -- run 方法观察者, springboot 应用启动步骤

1. SpringApplication#run 方法的观察者。
2. 每次 SpringApplication#run 方法应创建一个新的 SpringApplicationRunListener 实例。
3. 通过 SpringApplicationRunListeners 的生命周期函数，可以观察到 springboot 应用到启动步骤。

```java
public interface SpringApplicationRunListener {
    /**
     * Called immediately when the run method has first started. Can be used for very
     * early initialization.
     * SpringApplication#run 方法执行时立即调用。
     */
    void starting();

    /**
     * Called once the environment has been prepared, but before the
     * ApplicationContext has been created.
     * Environment 构建完成后，创建 ApplicationContext 前调用。
     */
    void environmentPrepared(ConfigurableEnvironment environment);

    /**
     * Called once the ApplicationContext has been created and prepared, but
     * before sources have been loaded.
     * 创建 ApplicationContext 之后，sources 加载前调用。
     */
    void contextPrepared(ConfigurableApplicationContext context);

    /**
     * Called once the application context has been loaded but before it has been
     * refreshed.
     * ApplicationContext sources 加载后，refresh 前调用。
     */
    void contextLoaded(ConfigurableApplicationContext context);

    /**
     * The context has been refreshed and the application has started but
     * CommandLineRunners and ApplicationRunners have not been called.
     * @since 2.0.0
     * ApplicationContext refresh 后，application 已启动，但尚未调用 CommandLineRunners 和 ApplicationRunners。
     */
    void started(ConfigurableApplicationContext context);

    /**
     * Called immediately before the run method finishes, when the application context has
     * been refreshed and all CommandLineRunners and ApplicationRunners have been called.
     * @since 2.0.0
     * ApplicationContext CommandLineRunners 和 ApplicationRunners 调用后, SpringApplication#run 方法完成前。
     */
    void running(ConfigurableApplicationContext context);

    /**
     * Called when a failure occurs when running the application.
     * @since 2.0.0
     * application 运行失败时调用
     */
    void failed(ConfigurableApplicationContext context, Throwable exception);
}
```

### getRunListeners

1. 根据 springFactories 机制获取 SpringApplicationRunListener 实现。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    // 根据 springFactories 机制获取 SpringApplicationRunListener 实现
    return new SpringApplicationRunListeners(logger,
            getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

默认情况下加载的 listeners 实现有一个 EventPublishingRunListener



## Environment -- 配置上下文

注意 Environment 是 spring-ioc 的概念，不是 springboot 独有的概念。

摘抄一段注释

> Interface representing the environment in which the current application is running. Models two key aspects of the application environment: profiles and properties. Methods related to property access are exposed via the `PropertyResolver`superinterface. A profile is a named, logical group of bean definitions to be registered with the container only if the given profile is active. Beans may be assigned to a profile whether defined in XML or via annotations; see the `spring-beans 3.1 schema` or the `@Profile` annotation for syntax details. The role of the `Environment`object with relation to profiles is in determining which profiles (if any) are currently active, and which profiles (if any) should be active by default. Properties play an important role in almost all applications, and may originate from a variety of sources: properties files, JVM system properties, system environment variables, JNDI, servlet context parameters, ad-hoc Properties objects, Maps, and so on. The role of the environment object with relation to properties is to provide the user with a convenient service interface for configuring property sources and resolving properties from them. Beans managed within an `ApplicationContext` may register to be `EnvironmentAware` or `@Inject` the Environment in order to query profile state or resolve properties directly. In most cases, however, application-level beans should not need to interact with the Environment directly but instead may have to have ${...} property values replaced by a property placeholder configurer such as `PropertySourcesPlaceholderConfigurer`, which itself is EnvironmentAware and as of Spring 3.1 is registered by default when using `<context:property-placeholder/>`. Configuration of the environment object must be done through the `ConfigurableEnvironment` interface, returned from all `AbstractApplicationContext` subclass `getEnvironment()` methods. See `ConfigurableEnvironment Javadoc` for usage examples demonstrating manipulation of property sources prior to application context `refresh()`.
>
> 表示当前应用程序正在其中运行的环境的接口。它为应用环境制定了两个关键的方面：**profile** 和 **properties**。与属性访问有关的方法通过 `PropertyResolver` 这个父接口公开。
>
> profile 机制保证了仅在指定 profile 处于激活状态时，才向容器注册的 Bean 定义的命名组。无论是用 XML 定义还是通过注解定义，都可以将 Bean 分配给指定的 profile。有关语法的详细信息，请参见 `spring-beans 3.1规范文档`  或 `@Profile` 注解。Environment 的作用是决定当前哪些配置文件（如果有）处于活动状态，以及默认情况下哪些配置文件（如果有）应处于活动状态。
>
> `Properties` 在几乎所有应用程序中都起着重要作用，并且可能来源自多种途径：属性文件，JVM系统属性，系统环境变量，JNDI，ServletContext 参数，临时属性对象，Map等。`Environment` 与 `Properties` 的关系是为用户提供方便的服务接口，以配置属性源，并从中解析属性值。
>
> 在 `ApplicationContext` 中管理的Bean可以注册为 `EnvironmentAware` 或使用 `@Inject` 标注在 Environment 上，以便直接查询 profile 的状态或解析 `Properties`。
>
> 在多数情况下，应用程序级 Bean 不必直接与 Environment 交互，而是通过将占位符 ${...} 属性值替换为属性占位符配置器进行属性注入（例如 `PropertySourcesPlaceholderConfigurer`），该属性本身是 `EnvironmentAware`，当配置了 `<context:property-placeholder/>` 时，默认情况下会使用Spring 3.1的规范注册。
>
> 必须通过从所有 `AbstractApplicationContext` 子类的 `getEnvironment()` 方法返回的 `ConfigurableEnvironment` 接口完成环境对象的配置。请参阅 `ConfigurableEnvironment` 的Javadoc以获取使用示例，这些示例演示在应用程序上下文 `refresh()` 方法被调用之前对属性源进行的操作。

简单概括一下: 它是 IOC 容器的运行环境，定义了 Profile 和 Properties 两部分模型。

1. properties: 配置内容 key-value。
2. profile: 可以按一个 profile 为一组(每个组多个 properties)，编写不同组的 properties, 根据需要激活一个或多个 profile, 激活的 profile 的 properties 可以在 ApplicationContext 及其管理的 bean 中获取。

### ConfigurableEnvironment -- setter 能力

1. Environment 提供 getter 能力, ConfigurableEnvironment 提供 setter 能力。getter, setter 接口分开的思想。
2. 与 ApplicationContext, ConfigurableApplicationContext 的 getter, setter 接口分开思想一样。

### 准备配置上下文 prepareEnvironment

org.springframework.boot.SpringApplication#prepareEnvironment

1. 创建和配置 environment
2. 调用 SpringApplicationRunListener, 时机是 environment 构建完成，ApplicationContext 创建之前。
3. environment 与 application 绑定。

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 创建 environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置 environment
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 回调 SpringApplicationRunListener, 时机是 environment 构建完成，ApplicationContext 创建之前。
    listeners.environmentPrepared(environment);
    // environment 与 application 绑定
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

#### 创建和配置 environment

org.springframework.boot.SpringApplication#getOrCreateEnvironment

1. 根据 application 类型创建不同的 Environment 实现。

```java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    // 根据 application 类型创建不同的 Environment 实现
    switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();
        case REACTIVE:
            return new StandardReactiveWebEnvironment();
        default:
            return new StandardEnvironment();
    }
}
```

org.springframework.boot.SpringApplication#configureEnvironment

1. 设置一个 ConversionService 提供类型转换能力。
2. 配置 PropertySource 和 Profiles，一些组合的杂活，没有做主要的事情。

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    // 设置一个 ConversionService 提供类型转换能力。
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    // 配置 PropertySource 和 Profiles，一些组合的杂活，没有做主要的事情。
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}
```

默认实现 DefaultConversionService 的组合属性:

```
StringToNumberConverterFactory
StringToBooleanConverter
IntegerToEnumConverterFactory
ArrayToCollectionConverter
StringToArrayConverter
```

#### environment 与 application 绑定

org.springframework.boot.SpringApplication#bindToSpringApplication

1. 把配置内容绑定到指定的属性配置类。

```java
/**
 * Bind the specified target Bindable using this binder's property sources.
 */
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
    try {
        Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
    }
    catch (Exception ex) {
        throw new IllegalStateException("Cannot bind to SpringApplication", ex);
    }
}

public <T> BindResult<T> bind(String name, Bindable<T> target) {
    return bind(ConfigurationPropertyName.of(name), target, null);
}

public <T> BindResult<T> bind(ConfigurationPropertyName name, Bindable<T> target, BindHandler handler) {
    Assert.notNull(name, "Name must not be null");
    Assert.notNull(target, "Target must not be null");
    handler = (handler != null) ? handler : BindHandler.DEFAULT;
    Context context = new Context();
    T bound = bind(name, target, handler, context, false);
    return BindResult.of(bound);
}
```

#### tips: 通过 Enviroment 从 application.properties 中读取 key-value

参考 DelegatingApplicationContextInitializer 做法:

1. 调用 ApplicationContext#getEnvironment 获得 ConfigurableEnvironment，再调用 Environment#getProperty 传入 key，即可得到 value。

```java
	private static final String PROPERTY_NAME = "context.initializer.classes";

	@Override
	public void initialize(ConfigurableApplicationContext context) {
		ConfigurableEnvironment environment = context.getEnvironment();
		List<Class<?>> initializerClasses = getInitializerClasses(environment);
		if (!initializerClasses.isEmpty()) {
			applyInitializerClasses(context, initializerClasses);
		}
	}

	private List<Class<?>> getInitializerClasses(ConfigurableEnvironment env) {
		String classNames = env.getProperty(PROPERTY_NAME);
		List<Class<?>> classes = new ArrayList<>();
		if (StringUtils.hasLength(classNames)) {
			for (String className : StringUtils.tokenizeToStringArray(classNames, ",")) {
				classes.add(getInitializerClass(className));
			}
		}
		return classes;
	}
```



## 设置系统参数 -- configureIgnoreBeanInfo

不重要，暂不关注。



## 打印 banner -- printBanner

banner 是这样的文字图案。

```


  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)


```



## 创建 ApplicationContext -- createApplicationContext

org.springframework.boot.SpringApplication#createApplicationContext

1. 根据 webApplicationType 实例化 ConfigurableApplicationContext 实例

```java
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 根据 webApplicationType 实例化 ConfigurableApplicationContext 实例
            switch (this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                    break;
                case REACTIVE:
                    contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                    break;
                default:
                    contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, " + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

到这里，根据 webApplicationType 创建的 environment 和 ApplicationContext 如下:

1. Servlet - StandardServletEnvironment - AnnotationConfigServletWebServerApplicationContext

2. Reactive - StandardReactiveWebEnvironment - AnnotationConfigReactiveWebServerApplicationContext

3. None - StandardEnvironment - AnnotationConfigApplicationContext

ApplicationContext 的父类构造方法中，beanFactory 实例化。

```java
public GenericApplicationContext() {
    this.beanFactory = new DefaultListableBeanFactory();
}
```



## 准备 ApplicationContext -- prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 向 ApplicationContext 设置 environment 引用
    context.setEnvironment(environment);
    // ApplicationContext 的发布处理
    postProcessApplicationContext(context);
    // 执行 Initializer
    applyInitializers(context);
    // 回调 SpringApplicationRunListener，时机是创建 ApplicationContext 后，sources 加载前。
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 向 beanFactory 注册两个组件：1. 在控制台打印 Banner 的 2. 之前把 main 方法中参数封装成对象的组件
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // Load the sources
    // 获取 sources(获取主启动类)
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 加载 source(加载主启动类)
    load(context, sources.toArray(new Object[0]));
    // 回调 SpringApplicationRunListener, 时机是创建 ApplicationContext 后，refresh 调用前。
    listeners.contextLoaded(context);
}
```

### ApplicationContext 的发布处理

1. 设置 resourceLoader 和 ClassLoader，之前准备好了。
2. 设置类型转换器 ConversionService，之前准备好了，并且还做了容器共享。

```java
public static final String CONFIGURATION_BEAN_NAME_GENERATOR =
			"org.springframework.context.annotation.internalConfigurationBeanNameGenerator";

protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    // 注册 BeanName 生成器
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                this.beanNameGenerator);
    }
    // 设置资源加载器和类加载器
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    // 设置类型转换器
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
    }
}
```

### 应用 ApplicationContextInitializer

遍历被添加进 SpringApplication 中的 ApplicationContextInitializer，执行 initialize 方法

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 遍历 ApplicationContextInitializer 执行
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```



## 获取 sources(获取主启动类) -- getAllSources

要获取 primarySources 和 sources。

primarySources 已经被设置过了，是 SpringApplication#run 方法所在的主启动类。sources 暂时不了解是什么，debug 会发现为空，暂不关注。

也就是说，getAllSources 能获取到主启动类。

```java
private Set<Class<?>> primarySources;
private Set<String> sources = new LinkedHashSet<>();

public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet<>();
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        allSources.addAll(this.primarySources);
    }
    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }
    return Collections.unmodifiableSet(allSources);
}
```



## 加载 sources(加载主启动类) -- load

```java
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    // 创建 BeanDefinitionLoader
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    // 设置 BeanName 生成器
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    // 设置资源加载器
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    // 设置运行环境
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}
```

### 创建 BeanDefinitionLoader

```java
	private BeanDefinitionRegistry getBeanDefinitionRegistry(ApplicationContext context) {
		if (context instanceof BeanDefinitionRegistry) {
			return (BeanDefinitionRegistry) context;
		}
		if (context instanceof AbstractApplicationContext) {
			return (BeanDefinitionRegistry) ((AbstractApplicationContext) context).getBeanFactory();
		}
		throw new IllegalStateException("Could not locate BeanDefinitionRegistry");
	}
```

使用组合模式，把 BeanDefinitionRegistry 和 sources 组合进了 BeanDefinitionLoader 中

三个关键的 Bean 解析器(准确来说是 BeanDefinition 解析器):

1. AnnotatedBeanDefinitionReader: 用于解析注解定义的 bean
2. XmlBeanDefinitionReader: 用于解析 xml 定义的 bean
3. ClassPathBeanDefinitionScanner: classpath 路径 bean 扫描器

```java
	protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {
        // 使用组合模式，把 BeanDefinitionRegistry 和 sources 组合进了 BeanDefinitionLoader 中
		return new BeanDefinitionLoader(registry, sources);
	}

	BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
		Assert.notNull(registry, "Registry must not be null");
		Assert.notEmpty(sources, "Sources must not be empty");
		this.sources = sources;
        // 注解定义的 beanDefinitionReader
		this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
        // xml 定义的 beanDefinitionReader
		this.xmlReader = new XmlBeanDefinitionReader(registry);
        // groovy 相关暂且不管
		if (isGroovyPresent()) {
			this.groovyReader = new GroovyBeanDefinitionReader(registry);
		}
        // classpath 路径 bean 扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(registry);
        // sources 类不计入 classpath 扫描
		this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
	}
```

### 使用 BeanDefinitionLoader 加载 sources(加载主启动类)

主启动类，它被 `@SpringBootApplication` 注解标注，而 `@SpringBootApplication` 组合了一个 `@SpringBootConfiguration`，它又组合了一个 `@Configuration` 注解，`@Configuration` 的底层就是一个 `@Component` 。

1. 判断 source 类是否是 Component(@Component 注解及其派生注解标注的类)。
   1. 扫描 @Component 注解时没有用反射的方式扫描注解，而是用类 asm 的方式扫描字节码。反射会触发类加载，spring 不想在 bean 还没有创建前进行类加载，类加载可能会触发 static 方法，static 方法中可能有 bean 自定义的业务逻辑。
2. 使用解析注解定义的 bean 用的 AnnotatedBeanDefinitionReader，注册 source

```java
	/**
	 * Load the sources into the reader.
	 * @return the number of loaded beans
	 */
	public int load() {
		int count = 0;
		for (Object source : this.sources) {
			count += load(source);
		}
		return count;
	}

	private int load(Object source) {
		Assert.notNull(source, "Source must not be null");
		if (source instanceof Class<?>) {
			return load((Class<?>) source);
		}
		if (source instanceof Resource) {
			return load((Resource) source);
		}
		if (source instanceof Package) {
			return load((Package) source);
		}
		if (source instanceof CharSequence) {
			return load((CharSequence) source);
		}
		throw new IllegalArgumentException("Invalid source type " + source.getClass());
	}

	private int load(Class<?> source) {
        // groovy 相关不管
		if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
			// Any GroovyLoaders added in beans{} DSL can contribute beans here
			GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
			load(loader);
		}
        // 判断是否是 Component(@Component 注解及其派生注解标注的类)
		if (isComponent(source)) {
            // 使用解析注解定义的 bean 用的 AnnotatedBeanDefinitionReader，注册 source
			this.annotatedReader.register(source);
			return 1;
		}
		return 0;
	}

	private boolean isComponent(Class<?> type) {
		// This has to be a bit of a guess. The only way to be sure that this type is
		// eligible is to make a bean definition out of it and try to instantiate it.
        // 注意这里没有用反射的方式扫描注解，而是用类 asm 的方式扫描字节码。
        // 反射会触发类加载，spring 不想在 bean 还没有创建前进行类加载，类加载可能会触发 static 方法，static 方法中可能有 bean 自定义的业务逻辑。
		if (AnnotationUtils.findAnnotation(type, Component.class) != null) {
			return true;
		}
		// Nested anonymous classes are not eligible for registration, nor are groovy
		// closures
		if (type.getName().matches(".*\\$_.*closure.*") || type.isAnonymousClass() || type.getConstructors() == null
				|| type.getConstructors().length == 0) {
			return false;
		}
		return true;
	}
```

### 解析注解定义的 bean -- AnnotatedBeanDefinitionReader

1. 创建 BeanDefinition
2. 解析 scope 作用域
3. 生成 beanName
4. 解析注解声明的 BeanDefinition 属性
5. 扩展点，允许使用 BeanDefinitionCustomizer 修改当前 BeanDefinition
6. 使用 BeanDefinitionHolder，将 BeanDefinition 注册到 beanRegistry 中

```java
    public void registerBean(Class<?> beanClass) {
        doRegisterBean(beanClass, null, null, null);
    }

	/**
	 * Register a bean from the given bean class, deriving its metadata from
	 * class-declared annotations.
	 */
	private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
			@Nullable BeanDefinitionCustomizer[] customizers) {
        // 创建 BeanDefinition
		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		abd.setInstanceSupplier(supplier);
        // 解析 scope 作用域
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
        // 生成 beanName
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

        // 解析注解声明的 BeanDefinition 属性
		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
		if (customizers != null) {
            // 扩展点，允许使用 BeanDefinitionCustomizer 修改当前 BeanDefinition
			for (BeanDefinitionCustomizer customizer : customizers) {
				customizer.customize(abd);
			}
		}

        // 使用 BeanDefinitionHolder，将 BeanDefinition 注册到 beanRegistry 中
		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```

### 将 BeanDefinition 注册到 beanRegistry 中

1. 将 beanName 和 aliases 分别向 beanRegistry 注册。

```java
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

### tips: springboot 主启动类上通过注解声明的 bean 如何被实例化的

1. springboot 主启动类先在 SpringApplication#run 方法里被注册为了一个 BeanDefinition。

2. ApplicationContext#refresh 时，执行的 finishBeanFactoryInitialization 方法会非延迟地初始化 bean 实例。主启动类的 BeanDefinition 会走 doCreateBean 的流程被实例化为一个 bean，doCreateBean 流程中，springboot 主启动类上面注解声明的 bean 被实例化了。
