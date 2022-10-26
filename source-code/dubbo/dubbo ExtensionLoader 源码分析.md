# dubbo ExtensionLoader 源码分析



## 说明

1. 本文基于 jdk 8, dubbo 2.7.18 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 类层次

向上继承、实现关系：

无继承无实现。

向上向下所属层次：

无子类。

类签名：

```java
/**
 * {@link org.apache.dubbo.rpc.model.ApplicationModel}, {@code DubboBootstrap} and this class are
 * at present designed to be singleton or static (by itself totally static or uses some static fields).
 * So the instances returned from them are of process or classloader scope. If you want to support
 * multiple dubbo servers in a single process, you may need to refactor these three classes.
 * <p>
 * Load dubbo extensions
 * <ul>
 * <li>auto inject dependency extension </li>
 * <li>auto wrap extension in wrapper </li>
 * <li>default extension is an adaptive instance</li>
 * </ul>
 *
 * @see <a href="http://java.sun.com/j2se/1.5.0/docs/guide/jar/jar.html#Service%20Provider">Service Provider in Java 5</a>
 * @see org.apache.dubbo.common.extension.SPI
 * @see org.apache.dubbo.common.extension.Adaptive
 * @see org.apache.dubbo.common.extension.Activate
 */
public class ExtensionLoader<T> {
```



## 关键属性

```java
    /**
     * static final 常量，extensionLoader 实例缓存，{SPI 接口 clazz - ExtensionLoader 对象} key-value pair
     */
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);

    /**
     * static final 常量，extension 实例缓存，存储 {extension clazz - extension 对象} key-value pair
     */
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>(64);

    /**
     * SPI 接口 clazz
     */
    private final Class<?> type;

    /**
     * 对象工厂，作为承载 extension 的容器
     */
    private final ExtensionFactory objectFactory;

    /**
     * extension name 缓存，key: extension clazz, value: extension name
     */
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();

    /**
     * extension clazz 缓存 pair，key: extension name，value: extension clazz
     */
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();

    private final Map<String, Object> cachedActivates = new ConcurrentHashMap<>();
	/**
     * extension 实例缓存，key: extension name, value: extension 实例
     */
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();

	/**
     * 接口对应实现类名称存储路径
     */
    private static volatile LoadingStrategy[] strategies = loadLoadingStrategies();
    /**
     * SPI 接口的 name
     */
	private String cachedDefaultName;
```

主要是一些缓存 ConcurrentMap，ExtensionLoader 和 extension 创建时的部分线程安全，借助于 ConcurrentMap 的原子化 api 保证。

这些缓存都比较关键，省略了一部分属性没有列出。



## 构造方法

```java
    /**
     * ExtensionLoader 构造方法
     *
     * @param type SPI 接口
     */
    private ExtensionLoader(Class<?> type) {
        // 在 ExtensionLoader 构造方法中指定 SPI 接口，用对象属性 type 表示
        this.type = type;
        // 创建 extensionFactory 的 adaptive 实例，或不创建
        objectFactory =
                (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

1. 在 ExtensionLoader 构造方法中指定 SPI 接口，用对象属性 type 表示。
2. 创建 extensionFactory 的 adaptive 实例，或不创建。



## getExtensionLoader 方法

通过接口 clazz，获得对应的 extensionLoader 对象

```java
    /**
     * 通过接口 clazz，获得对应的 extensionLoader 对象
     *
     * @param type
     * @param <T>
     * @return
     */
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        // 边界条件处理
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }

        // 尝试从 extensionLoader 实例缓存中获取
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        // extensionLoader 实例缓存中没有，则创建
        if (loader == null) {
            /*
             * 使用原子化 api 添加至缓存中。当 key 不存在时，作为回调函数触发 new 一个 extensionLoader 作为 value。
             * 先用原子化 api put，然后再 get value，而不是直接 new 一个 extensionLoader 返回。这里借助原子化 api 消除了竟态条件。
             */
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

1. 边界条件处理。
2. 尝试从 extensionLoader 实例缓存中获取。
3. extensionLoader 实例缓存中没有，则创建。
4. 使用原子化 api 添加至缓存中。当 key 不存在时，作为回调函数触发 new 一个 extensionLoader 作为 value。先用原子化 api put，然后再 get value，而不是直接 new 一个 extensionLoader 返回。这里借助原子化 api 消除了竟态条件。



## getExtension 方法

org.apache.dubbo.common.extension.ExtensionLoader#getExtension(java.lang.String, boolean)

获取 extension，如果 extension 不存在，会进行创建。

```java
    /**
     * 获取 extension，如果 extension 不存在，会进行创建
     *
     * @param name
     * @param wrap
     * @return
     */
    public T getExtension(String name, boolean wrap) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            // 获取默认的 extension 实例
            return getDefaultExtension();
        }
        // 使用 holder 用于持有目标 extension 实例，并在 extension 实例缓存中根据 name 获取
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        // 如果 extension 实例不存在，则创建。双检锁保证创建时线程安全
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    // 在线程安全区域，创建 extension
                    instance = createExtension(name, wrap);
                    // 放回对象载体 holder 中
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

1. 如果 extension name 传入一个"true"，则获取默认的 extension 实例。
2. 使用 holder 用于持有目标 extension 实例，并在 extension 实例缓存中根据 name 获取。
3. 如果 extension 实例不存在，则创建。双检锁保证创建时线程安全。
4. 在线程安全区域，创建 extension，放回对象载体 holder 中。



## getOrCreateHolder 方法

使用 holder 用于持有目标 extension 实例，并在 extension 实例缓存中根据 name 获取。

```java
    /**
     * 使用 holder 用于持有目标 extension 实例，并在 extension 实例缓存中根据 name 获取
     *
     * @param name 目标 extension name
     * @return
     */
    private Holder<Object> getOrCreateHolder(String name) {
        // 在 extension 实例缓存中根据 name 获取
        Holder<Object> holder = cachedInstances.get(name);
        // extension 实例未找到，则创建一个 holder 用于持有目标 extension
        if (holder == null) {
            // holder 创建时使用 ConcurrentMap#putIfAbsent 原子化 api 保证线程安全
            cachedInstances.putIfAbsent(name, new Holder<>());
            // 多线程创建同一个 name 对应的 holder 时，可能其他线程已经创建，再尝试获取一次
            holder = cachedInstances.get(name);
        }
        // 返回 holder
        return holder;
    }
```

1. 在 extension 实例缓存中根据 name 获取。
2. extension 实例未找到，则创建一个 holder 用于持有目标 extension。
3. holder 创建时使用 ConcurrentMap#putIfAbsent 原子化 api 保证线程安全。
4. 多线程创建同一个 name 对应的 holder 时，可能其他线程已经创建，再尝试获取一次。
5. 返回 holder。



## createExtension 方法

创建 extension

```java
    /**
     * 创建 extension
     *
     * @param name
     * @param wrap
     * @return
     */
    @SuppressWarnings("unchecked")
    private T createExtension(String name, boolean wrap) {
        // 从 extension clazz 缓存中获取 clazz
        Class<?> clazz = getExtensionClasses().get(name);
        // 边界条件处理
        if (clazz == null || unacceptableExceptions.contains(name)) {
            throw findException(name);
        }
        try {
            // 尝试从本地缓存中加载
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                // 缓存中没有 extension 实例，原子化创建
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.getDeclaredConstructor().newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 给指定的实例注入 extension
            injectExtension(instance);

            // 是否是装饰类
            if (wrap) {
                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                // getExtensionClasses -> loadExtensionClasses 中如果是装饰类，便会将其放入 cachedWrapperClasses，那么此处 wrapperClassesList 就不会为空
                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        if (wrapper == null
                                || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                        }
                    }
                }
            }

            // 初始化 extension
            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

1. 从 extension clazz 缓存中获取 clazz。
2. 尝试从本地缓存中加载。
3. 缓存中如果没有 extension 实例，使用 ConcurrentHashMap#putIfAbsent 方法原子化创建。
4. 给指定的实例注入 extension。
5. 装饰类相关逻辑。
6. 初始化 extension。



## injectExtension 方法

给指定的实例注入 extension
具体地说，指定的 instance 属性如果有 extension，此方法将会给这些属性注入 extension

```java
    /**
     * 给指定的实例注入 extension
     * 具体地说，指定的 instance 属性如果有 extension，此方法将会给这些属性注入 extension
     *
     * @param instance
     * @return
     */
    private T injectExtension(T instance) {
        // 边界条件处理
        if (objectFactory == null) {
            return instance;
        }

        try {
            // 遍历实例的所有方法
            for (Method method : instance.getClass().getMethods()) {
                // 只放行 setter 方法
                if (!isSetter(method)) {
                    continue;
                }

                /*
                 * Check {@link DisableInject} to see if we need autowire injection for this property
                 * 加了 @DisableInject 注解的方法不放行
                 */
                if (method.getAnnotation(DisableInject.class) != null) {
                    continue;
                }

                // 检查 setter 入参的数据类型
                Class<?> pt = method.getParameterTypes()[0];
                // 基础类型不放行。因为 injectExtension 负责给指定的实例注入 extension，基础数据类型肯定不是 extension。
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }

                /*
                 * Check {@link Inject} to see if we need auto-injection for this property
                 * {@link Inject#enable} == false will skip inject property phase
                 * {@link Inject#InjectType#ByName} default inject by name
                 */
                // setter 要设置的 extension 的 name
                String property = getSetterProperty(method);
                // 检查方法的 @Inject 注解，此注解用于控制是否要进行注入，以及 byType 还是 byName 进行注入
                Inject inject = method.getAnnotation(Inject.class);
                // 如果 setter 没有 @Inject 注解，默认 byName 进行注入
                if (inject == null) {
                    injectValue(instance, method, pt, property);
                } else {
                    // @Inject 注解只放行 enable
                    if (!inject.enable()) {
                        continue;
                    }

                    // byType，通过 extension 的 type 注入
                    if (inject.type() == Inject.InjectType.ByType) {
                        injectValue(instance, method, pt, null);
                    } else {
                        // byName，通过 extension 的 name 进行注入
                        injectValue(instance, method, pt, property);
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        // 返回注入 extension 完成的 instance
        return instance;
    }
```

1. 遍历实例的所有方法。
2. 只放行 setter 方法。
3. 加了 @DisableInject 注解的方法不放行。
4. 检查 setter 入参的数据类型。基础类型不放行。因为 injectExtension 负责给指定的实例注入 extension，基础数据类型肯定不是 extension。
5. 获取 setter 要设置的 extension 的 name。
6. 检查方法的 @Inject 注解，此注解用于控制是否要进行注入，以及 byType 还是 byName 进行注入。
7. 如果 setter 没有 @Inject 注解，默认 byName 进行注入。
8. @Inject 注解只放行 enable。
9. byType，通过 extension 的 type 注入。byName，通过 extension 的 name 进行注入。
10. 返回注入 extension 完成的 instance。



## injectValue 方法

真正地给指定的实例注入 extension

```java
    /**
     * 真正地给指定的实例注入 extension
     *
     * @param instance
     * @param method 传入这里的 method 肯定是一个 setter 方法，这是前置方法 injectExtension 保证的
     * @param pt
     * @param property
     */
    private void injectValue(T instance, Method method, Class<?> pt, String property) {
        try {
            /*
             * type 和 name 一起传入 ExtensionFactory#getExtension 方法获取 extension 实例。
             * ExtensionFactory#getExtension 方法由 ExtensionFactory 子类实现。
             */
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                /*
                 * 传入这里的 method 肯定是一个 setter 方法，这是前置方法 injectExtension 保证的。
                 * 调用 setter 方法，把 extension 实例设置给 instance 的属性
                 */
                method.invoke(instance, object);
            }
        } catch (Exception e) {
            logger.error("Failed to inject via method " + method.getName()
                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
        }
    }
```

1. type 和 name 一起传入 ExtensionFactory#getExtension 方法获取 extension 实例。
2. ExtensionFactory#getExtension 方法由 ExtensionFactory 子类实现。
3. 传入这里的 method 肯定是一个 setter 方法，这是前置方法 injectExtension 保证的。
4. 调用 setter 方法，把 extension 实例设置给 instance 的属性。



## getExtensionClasses 方法

获取 extension clazz

```java
    /**
     * 获取 extension clazz
     *
     * @return
     */
    private Map<String, Class<?>> getExtensionClasses() {
        // 从缓存中加载 extension clazz
        Map<String, Class<?>> classes = cachedClasses.get();
        // 缓存中没有，则加载 extension clazz。双检锁保证创建时线程安全。
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    // 加载 extension clazz
                    classes = loadExtensionClasses();
                    // 加载的 extension clazz 放入缓存
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

1. 从缓存中加载 extension clazz。
2. 缓存中没有，则加载 extension clazz。双检锁保证创建时线程安全。
3. 调用 loadExtensionClasses 方法，加载 extension clazz。
4. 加载的 extension clazz 放入缓存。



## loadExtensionClasses 方法

org.apache.dubbo.common.extension.ExtensionLoader#loadExtensionClasses

加载 extension clazz。

此方法外通过 synchronized 双检锁保证了线程安全，此方法内无需关心线程安全问题。

```java
    /**
     * synchronized in getExtensionClasses
     *
     * 加载 extension clazz
     * 此方法外通过 synchronized 双检锁保证了线程安全，此方法内无需关心线程安全问题
     */
    private Map<String, Class<?>> loadExtensionClasses() {
        // 获取 SPI 接口上的 @SPI 注解，并且缓存 SPI 接口的 name
        cacheDefaultExtensionName();

        // 由于 loadExtensionClasses 方法外使用 synchronized 双检锁保证了线程安全，因此这里使用性能更高的 HashMap，而不是 ConcurrentHashMap
        Map<String, Class<?>> extensionClasses = new HashMap<>();

        /*
         * 加载策略，去指定路径进行加载。策略模式
         */
        for (LoadingStrategy strategy : strategies) {
            // 去指定路径加载 extension clazz
            loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(),
                    strategy.overridden(), strategy.excludedPackages());
            // 历史原因，dubbo 项目加入 org.apache 之前，所在包是 com.alibaba。接口全限定名都改成 com.alibaba 开头，再次尝试加载。
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"),
                    strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
```

1. 获取 SPI 接口上的 @SPI 注解，并且缓存 SPI 接口的 name
2. 由于 loadExtensionClasses 方法外使用 synchronized 双检锁保证了线程安全，因此这里使用性能更高的 HashMap，而不是 ConcurrentHashMap。
3. 遍历加载策略，去指定路径进行加载。策略模式。
4. 去指定路径加载 extension clazz。
5. 历史原因，dubbo 项目加入 org.apache 之前，所在包是 com.alibaba。接口全限定名都改成 com.alibaba 开头，再次尝试加载。



## cacheDefaultExtensionName 方法

获取 SPI 接口上的 @SPI 注解，并且缓存 SPI 接口的 name。

```java
    /**
     * extract and cache default extension name if exists
     *
     * 获取 SPI 接口上的 @SPI 注解，并且缓存 SPI 接口的 name
     */
    private void cacheDefaultExtensionName() {
        // 获取 SPI 接口上的 @SPI 注解
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation == null) {
            return;
        }

        // 拿到 @SPI 注解的 value
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            // SPI 接口只能有一个 name
            if (names.length > 1) {
                throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) {
                // 将 SPI 接口的 name 赋值给 cachedDefaultName
                cachedDefaultName = names[0];
            }
        }
    }
```

1. 获取 SPI 接口上的 @SPI 注解。
2. 拿到 @SPI 注解的 value。
3. SPI 接口只能有一个 name。
4. 将 SPI 接口的 name 赋值给 cachedDefaultName。



## loadDirectory 方法

org.apache.dubbo.common.extension.ExtensionLoader#loadDirectory(java.util.Map<java.lang.String,java.lang.Class<?>>, java.lang.String, java.lang.String, boolean, boolean, java.lang.String...)

去指定路径加载 extension clazz

```java
    /**
     * 去指定路径加载 extension clazz
     *
     * @param extensionClasses
     * @param dir
     * @param type
     * @param extensionLoaderClassLoaderFirst
     * @param overridden
     * @param excludedPackages
     */
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                               boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
        // 将相对路径和 SPI 接口的全限定类名作为文件路径。例如: META-INF/services/dubbo.extension.demo.ClearStrategy
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls = null;
            // 寻找用于加载的 ClassLoader
            ClassLoader classLoader = findClassLoader();

            /*
             * try to load from ExtensionLoader's ClassLoader first
             * 尝试先通过 ExtensionLoader 的类加载器获得文件 url。一个开关属性控制，该属性通常由 LoaderStrategy 接口中一个 default 方法返回，并且为 false
             */
            if (extensionLoaderClassLoaderFirst) {
                // 获得 ExtensionLoader 的 classLoader
                ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
                // 如果不是系统使用的 classLoader
                if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                    // 通过 ExtensionLoader 的 classLoader 获取文件 url
                    urls = extensionLoaderClassLoader.getResources(fileName);
                }
            }

            // 如果 url 为空，则通过之前找到的 classLoader 获取文件 url
            if (urls == null || !urls.hasMoreElements()) {
                if (classLoader != null) {
                    // 如果之前找到 classLoader，根据文件路径获取文件 url
                    urls = classLoader.getResources(fileName);
                } else {
                    // 如果之前未找到 classLoader，使用系统的 classLoader 获取文件 url
                    urls = ClassLoader.getSystemResources(fileName);
                }
            }

            if (urls != null) {
                // 开始加载
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    // 解析指定文件，加载 extension 的 clazz
                    loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

1. 将相对路径和 SPI 接口的全限定类名作为文件路径。例如: META-INF/services/dubbo.extension.demo.ClearStrategy。
2. 寻找用于加载的 ClassLoader。
3. 尝试先通过 ExtensionLoader 的类加载器获得文件 url。一个开关属性控制，该属性通常由 LoaderStrategy 接口中一个 default 方法返回，并且为 false。
4. 如果 url 为空，则通过之前找到的 classLoader 获取文件 url。
5. 解析指定文件，加载 extension 的 clazz。