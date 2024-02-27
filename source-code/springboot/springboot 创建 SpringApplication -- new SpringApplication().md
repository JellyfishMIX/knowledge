# springboot 创建 SpringApplication -- new SpringApplication()



## 说明

1. 本文基于 springboot 2.1.x 编写
2. @author [blog.jellyfishmix.com](http://blog.jellyfishmix.com) / [JellyfishMIX - github](https://github.com/JellyfishMIX)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## SpringApplication

### SpringApplication demo

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### SpringApplication.run -- spring boot 启动

1. 第一步 new SpringApplication, 第二步执行 SpringApplication#run 方法。

```java
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        // 调用重载的方法
        return run(new Class<?>[] { primarySource }, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return new SpringApplication(primarySources)
            .run(args);
    }
```

### 创建 SpringApplication

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // resourceLoader 为 null
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 将传入的 DemoApplication 启动类放入 primarySources 中，这样应用就知道主启动类在哪里，叫什么了
    // springboot 一般称呼这种主启动类叫 primarySource(主配置资源来源)
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 判断当前应用环境
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 设置初始化器
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 确定主配置类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 自定义 SpringApplication 方式

如下 demo 可自定义 SpringApplication 实例。

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(DemoApplication.class);
        springApplication.setWebApplicationType(WebApplicationType.SERVLET); //强制使用WebMvc环境
        springApplication.setBannerMode(Banner.Mode.OFF); //不打印Banner
        springApplication.run(args);
    }
}
```



## ApplicationContextInitializer

```java
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext>
```

### 设置 ApplicationContextInitializer -三种方式设置

#### 方式一 手动向 SpringApplication 添加一个 ApplicationContextInitializer

原理: 没有原理，简单朴素地手动往 SpringApplication 里添加。

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(DemoApplication.class);
        // 手动加一个 ApplicationContextInitializer
        springApplication.addInitializers(new ApplicationContextInitializerDemo());
        springApplication.run(args);
    }
}

public class ApplicationContextInitializerDemo implements ApplicationContextInitializer {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        System.out.println("ApplicationContextInitializerDemo#initialize run...");
    }
}
```

#### 方式二 spring.factories 中配置接口的实现

原理: 通过 getSpringFactoriesInstances 方法使用了 springFactories 机制实现的。

```
org.springframework.context.ApplicationContextInitializer=com.example.demo.ApplicationContextInitializerDemo
```

#### 方式三 application.properties 中配置

在 application.properties 中配置如下内容

```properties
context.initializer.classes=com.example.demo.ApplicationContextInitializerDemo
```

原理: 通过 spring.factories 中默认 ApplicationContextInitializer 实现类之一 DelegatingApplicationContextInitializer 来加载的配置在 application.properties 中配置的 ApplicationContextInitializer 实现类。需要在 SpringApplication#run 执行时，才能把 application.properties 中的实现类加载并应用到 ApplicationContext，而不是 new SpringApplication 时。可以参考 DelegatingApplicationContextInitializer 实现类的代码。

### 默认的 ApplicationContextInitializer

```java
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // ...
    // 设置初始化器
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	// ...
}

public void setInitializers(Collection<? extends ApplicationContextInitializer<?>> initializers) {
    this.initializers = new ArrayList<>();
    this.initializers.addAll(initializers);
}
```

spring-boot 包 spring.factories:

```
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
```

spring-boot-autoconfigure 包 spring.factories:

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```

默认配置的 ApplicationContextInitialize 作用:

1. ConfigurationWarningsApplicationContextInitializer: 报告 IOC 容器的一些常见的错误配置。

2. ContextIdApplicationContextInitializer:: 设置 spring 应用上下文的 id。

3. DelegatingApplicationContextInitializer: 加载 application.properties 中 context.initializer.classes 配置的类。

4. ServerPortInfoApplicationContextInitializer: 将内置servlet容器实际使用的监听端口写入到 Environment 环境属性中。

5. SharedMetadataReaderFactoryContextInitializer: 创建一个 SpringBoot 和 ConfigurationClassPostProcessor 共用的 CachingMetadataReaderFactory 对象。

6. ConditionEvaluationReportLoggingListener: 将 `ConditionEvaluationReport` 写入日志



## ApplicationListener

### 设置 ApplicationListener

ApplicationListener 是 springframework 中的模型，用于监听 ApplicationContext 刷新过程中的事件。

三种方式设置与 ApplicationContextInitializer 一致: 手动添加，application.properties 中配置，spring.factories 中配置。

### ApplicationListener 的默认实现

spring-boot 包 spring.factories:

```
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.context.logging.LoggingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
```

spring-boot-autoconfigure 包 spring.factories:

```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```

默认配置的 ApplicationListener:

1. ClearCachesApplicationListener: 应用上下文加载完成后对缓存做清除工作

2. ParentContextCloserApplicationListener: 监听双亲应用上下文的关闭事件并往自己的子应用上下文中传播

3. FileEncodingApplicationListener: 检测系统文件编码与应用环境编码是否一致，如果系统文件编码和应用环境的编码不同则终止应用启动

4. AnsiOutputApplicationListener: 根据 spring.output.ansi.enabled 参数配置 AnsiOutput

5. ConfigFileApplicationListener: 从常见的那些约定的位置读取配置文件

6. DelegatingApplicationListener: 监听到事件后转发给 application.properties 中配置的 context.listener.classes 的监听器

7. ClasspathLoggingApplicationListener: 对环境就绪事件 ApplicationEnvironmentPreparedEvent 和应用失败事件  ApplicationFailedEvent 做出响应

8. LoggingApplicationListener: 配置 LoggingSystem。使用 logging.config 环境变量指定的配置或者缺省配置

9. LiquibaseServiceLocatorApplicationListener: 使用一个可以和 SpringBoot 可执行jar包配合工作的版本替换 LiquibaseServiceLocator

10. BackgroundPreinitializer: 使用一个后台线程尽早触发一些耗时的初始化任务



## springFactories 机制(类 SPI 机制) -- 从 springFactory 中获取 bean 实例

org.springframework.boot.SpringApplication#getSpringFactoriesInstances

1. 读取 spring.factories 文件，获取指定 clazz 配置的的实现类全限定名。spring.factories 文件配置格式是 key=instance0Refrence,instance1Refrence,instance2Refrence
2. 根据实现类全限定名，创建这些实现类的实例。
   1. 根据全限定名获取 clazz 对象，根据 clazz 使用反射实例化。

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates (使用 set 确保名称唯一，以防止重复)
    // 读取 spring.factories 文件，获取指定 clazz 配置的的实现类全限定名
    // spring.factories 文件配置格式是 key=instance0Refrence,instance1Refrence,instance2Refrence
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 根据实现类全限定名，创建这些实现类的实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}

private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
        ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            // 根据全限定名获取 clazz 对象
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            // 根据 clazz 使用反射实例化
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

