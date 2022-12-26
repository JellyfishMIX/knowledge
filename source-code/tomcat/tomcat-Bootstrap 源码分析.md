# tomcat-Bootstrap 源码分析



## Bootstrap#main 方法

Bootstrap#main 方法主要就做了三件事，首先类加载一顿，第二步各种组件嵌套 init 方法调用一顿，第三步各种组件嵌套 start 方法调用一顿。嵌套调用的时候大量应用反射调用 + 模版方法。



## Bootstrap#init() 方法

org.apache.catalina.startup.Bootstrap#init()

初始化 catalinaDaemon(Catalina 类的实例)。

1. 初始化类加载器。会创建三种类加载器，catalinaLoader 与 sharedLoader 的父类加载器都是 commonLoader。
2. 设置当前线程的上下文类加载器为 catalinaLoader
3. 使用 catalinaLoader 类加载器，加载 coyote, tomcat, javax 等各包的类。
4. 使用 catalinaLoader 加载 org.apache.catalina.startup.Catalina 类，通过反射实例化。
5. 通过反射调用 Catalina#setParentClassLoader 方法，把 sharedLoader 设置为 catalina 实例的父加载器。
6. catalina 实例赋值给 Bootstrap 类的成员属性 catalinaDaemon。

```java
    /**
     * Initialize daemon.
     *
     * 初始化 catalinaDaemon
     *
     * @throws Exception Fatal initialization error
     */
    public void init() throws Exception {
        // 初始化类加载器。会创建三种类加载器，catalinaLoader 与 sharedLoader 的父类加载器都是 commonLoader。
        initClassLoaders();
        // 设置当前线程的上下文类加载器为 catalinaLoader
        Thread.currentThread().setContextClassLoader(catalinaLoader);
        // 使用 catalinaLoader 类加载器，加载 coyote, tomcat, javax 等各包的类。
        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        if (log.isDebugEnabled()) {
            log.debug("Loading startup class");
        }
        // 使用 catalinaLoader 加载 org.apache.catalina.startup.Catalina 类，通过反射实例化
        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.getConstructor().newInstance();

        // Set the shared extensions class loader
        if (log.isDebugEnabled()) {
            log.debug("Setting startup class properties");
        }
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);
        // 通过反射调用 Catalina#setParentClassLoader 方法，把 sharedLoader 设置为 catalina 实例的父加载器。
        method.invoke(startupInstance, paramValues);
        // catalina 实例赋值给 Bootstrap 类的成员属性 catalinaDaemon
        catalinaDaemon = startupInstance;
    }
```

### Bootstrap#initClassLoaders 方法

org.apache.catalina.startup.Bootstrap#initClassLoaders

初始化类加载器，会创建三种类加载器，catalinaLoader 与 sharedLoader 的父类加载器都是 commonLoader。

```java
    /**
     * 初始化类加载器
     * 会创建三种类加载器，catalinaLoader 与 sharedLoader 的父类加载器都是 commonLoader。
     */
    private void initClassLoaders() {
        try {
            commonLoader = createClassLoader("common", null);
            if (commonLoader == null) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader = this.getClass().getClassLoader();
            }
            catalinaLoader = createClassLoader("server", commonLoader);
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            handleThrowable(t);
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
    }
```



## Bootstrap#load 方法

org.apache.catalina.startup.Bootstrap#load

加载 server 实例。通过反射调用 Catalina#load() 方法。

```java
    /**
     * Load daemon.
     *
     * 加载 server 实例
     */
    private void load(String[] arguments) throws Exception {

        // Call the load() method
        String methodName = "load";
        Object param[];
        Class<?> paramTypes[];
        if (arguments==null || arguments.length==0) {
            paramTypes = null;
            param = null;
        } else {
            paramTypes = new Class[1];
            paramTypes[0] = arguments.getClass();
            param = new Object[1];
            param[0] = arguments;
        }
        Method method =
            catalinaDaemon.getClass().getMethod(methodName, paramTypes);
        if (log.isDebugEnabled()) {
            log.debug("Calling startup class " + method);
        }
        // 通过反射调用 Catalina#load() 方法
        method.invoke(catalinaDaemon, param);
    }
```

### Catalina#load() 方法

org.apache.catalina.startup.Catalina#load()

加载一个 server 实例。

1. 创建 Digester 实例(server.xml 配置文件解析器)
2. 使用 digester 解析配置文件，创建 server 实例。
3. server 实例绑定当前 catalina 实例，设置当前 catalina 的根路径位置。
4. 调用 server#init 方法进行初始化，server 实例默认是 StandardServer。应用了模版方法模式，实际调用的是 LifecycleBase#init() 模版方法。

```java
/**
     * Start a new server instance.
     *
     * 加载一个 server 实例
     */
    public void load() {
        if (loaded) {
            return;
        }
        loaded = true;
        long t1 = System.nanoTime();
        initDirs();
        // Before digester - it may be needed
        initNaming();

        // Create and execute our Digester
        // 创建 Digester 实例(server.xml 配置文件解析器)
        Digester digester = createStartDigester();

        InputSource inputSource = null;
        InputStream inputStream = null;
        File file = null;
        try {
            try {
                file = configFile();
                inputStream = new FileInputStream(file);
                inputSource = new InputSource(file.toURI().toURL().toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail", file), e);
                }
            }
            // 省略部分代码
            try {
                inputSource.setByteStream(inputStream);
                digester.push(this);
                // 使用 digester 解析配置文件，创建 server 实例。
                digester.parse(inputSource);
            } catch (SAXParseException spe) {
                log.warn("Catalina.start using " + getConfigFile() + ": " +
                        spe.getMessage());
                return;
            } catch (Exception e) {
                log.warn("Catalina.start using " + getConfigFile() + ": " , e);
                return;
            }
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    // Ignore
                }
            }
        }
        // server 实例绑定当前 catalina 实例，设置当前 catalina 的根路径位置
        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());
        // Stream redirection
        initStreams();
        // Start the new server
        try {
            /*
             * 调用 server#init 方法进行初始化。server 实例默认是 StandardServer
             * 应用了模版方法模式，实际调用的是 LifecycleBase#init() 模版方法
             */
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new java.lang.Error(e);
            } else {
                log.error("Catalina.start", e);
            }
        }
        // 省略收尾部分代码
    }
```

### LifecycleBase#init 方法

org.apache.catalina.util.LifecycleBase#init

LifecycleBase 初始化，应用了模版方法模式，关键的 initInternal() 方法由子类实现。Server 接口的默认实现是 StandardServer。

1. 更新 LifecycleBase 的 state 为 INITIALIZING，触发 LifecycleEvent 生命周期事件。
2. 调用 LifecycleBase 的具体实现进行初始化。如果是 Server 接口的默认实现 StandardServer, 调用的是 StandardServer#initInternal()
3. 更新 LifecycleBase 的 state 为 INITIALIZED，触发 LifecycleEvent 生命周期事件。

```java
    /**
     * LifecycleBase 初始化，应用了模版方法模式，关键的 initInternal() 方法由子类实现。
     * Server 接口的默认实现是 StandardServer
     */
    @Override
    public final synchronized void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }

        try {
            // 更新 LifecycleBase 的 state 为 INITIALIZING，触发 LifecycleEvent 生命周期事件。
            setStateInternal(LifecycleState.INITIALIZING, null, false);
            // 调用 LifecycleBase 的具体实现进行初始化。如果是 Server 接口的默认实现 StandardServer, 调用的是 StandardServer#initInternal()
            initInternal();
            // 更新 LifecycleBase 的 state 为 INITIALIZED，触发 LifecycleEvent 生命周期事件。
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }
```

### StandardServer#initInternal 方法

org.apache.catalina.core.StandardServer#initInternal

初始化 server 实例。

1. 按照规范首先调用父类 LifecycleMBeanBase#initInternal 方法，有统一逻辑执行。
2. 注册全局字符串缓存，如果有多个 Server 则会注册多个字符串缓存对象。注册 MBeanFactory。注册全局命名资源并初始化，该全局命名资源配置在 server.xml 中。
3. 使用 catalina 的父加载器校验系统 JAR 包文件。
4. 调用 Service#init() 方法，初始化自定义的 service。

```java
    /**
     * Invoke a pre-startup initialization. This is used to allow connectors
     * to bind to restricted ports under Unix operating environments.
     *
     * 初始化 server 实例
     */
    @Override
    protected void initInternal() throws LifecycleException {
        // 调用父类 LifecycleMBeanBase#initInternal 方法
        super.initInternal();

        // Register global String cache
        // Note although the cache is global, if there are multiple Servers
        // present in the JVM (may happen when embedding) then the same cache
        // will be registered under multiple names
        // 注册全局字符串缓存，如果有多个 Server 则会注册多个字符串缓存对象
        onameStringCache = register(new StringCache(), "type=StringCache");

        // Register the MBeanFactory
        // 注册 MBeanFactory
        MBeanFactory factory = new MBeanFactory();
        factory.setContainer(this);
        onameMBeanFactory = register(factory, "type=MBeanFactory");

        // Register the naming resources
        // 注册全局命名资源并初始化，该全局命名资源配置在 server.xml 中
        globalNamingResources.init();

        // Populate the extension validator with JARs from common and shared
        // class loaders
        // 使用 catalina 的父加载器校验系统 JAR 包文件
        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            // Walk the class loader hierarchy. Stop at the system class loader.
            // This will add the shared (if present) and common class loaders
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File (url.toURI());
                                if (f.isFile() &&
                                        f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException | IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        // 初始化自定义的 service
        for (Service service : services) {
            service.init();
        }
    }
```

### StandardService#initInternal 方法

org.apache.catalina.core.StandardService#initInternal

初始化自定义的 service: 初始化关联的 executor, mapperListener, 所有关联的 connector

```java
    /**
     * Invoke a pre-startup initialization. This is used to allow connectors
     * to bind to restricted ports under Unix operating environments.
     *
     * 初始化自定义的 service
     */
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (engine != null) {
            engine.init();
        }

        // Initialize any Executors
        // 初始化关联的 executor
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // Initialize mapper listener
        // 初始化 mapper 监听器
        mapperListener.init();

        // Initialize our defined Connectors
        // 初始化所有关联的 connector 连接器
        synchronized (connectorsLock) {
            for (Connector connector : connectors) {
                try {
                    connector.init();
                } catch (Exception e) {
                    String message = sm.getString(
                            "standardService.connector.initFailed", connector);
                    log.error(message, e);

                    if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                        throw new LifecycleException(message);
                    }
                }
            }
        }
    }
```

### MapperListener#startInternal 方法

org.apache.catalina.mapper.MapperListener#startInternal

初始化 mapperListener

1. 设置 mapperListener 的 LifecycleState 为 STARTING
2. 查找默认的 host，绑定 mapperListener 到 engine 及子容器。
3. 注册 engine 容器到绑定的 host 以及上下文 context, wrapper。

```java
    /**
     * 初始化 mapperListener
     */
    @Override
    public void startInternal() throws LifecycleException {
        // 设置 mapperListener 的 LifecycleState 为 STARTING
        setState(LifecycleState.STARTING);

        Engine engine = service.getContainer();
        if (engine == null) {
            return;
        }

        // 查找默认的 host
        findDefaultHost();
        // 绑定 mapperListener 到 engine 及子容器
        addListeners(engine);

        Container[] conHosts = engine.findChildren();
        for (Container conHost : conHosts) {
            Host host = (Host) conHost;
            if (!LifecycleState.NEW.equals(host.getState())) {
                // Registering the host will register the context and wrappers
                // 注册 engine 容器到绑定的 host 以及上下文 context, wrapper。
                registerHost(host);
            }
        }
    }
```

### Connector#initInternal 方法

org.apache.catalina.connector.Connector#initInternal

connector 连接器初始化

1. 创建并初始化 coyote 适配器，关联至协议处理器 protocolHandler 中。
2. apr 协议处理器边界条件处理。ARP(Address Resolution Protocol)，地址解析协议，根据 ip 地址获取物理地址的一个 TCP/IP 协议。判断 apr 协议处理器是否开启 SSL。
3. 协议处理器执行初始化(protocolHandler 是在 server.xml 解析时创建的)。

```java
    /**
     * connector 连接器初始化
     */
    @SuppressWarnings("deprecation")
    @Override
    protected void initInternal() throws LifecycleException {

        super.initInternal();

        // Initialize adapter
        // 创建并初始化 coyote 适配器，关联至协议处理器 protocolHandler 中
        adapter = new CoyoteAdapter(this);
        protocolHandler.setAdapter(adapter);

        // Make sure parseBodyMethodsSet has a default
        // parseBodyMethodsSet 赋默认值，默认值为 POST 方法
        if (null == parseBodyMethodsSet) {
            setParseBodyMethods(getParseBodyMethods());
        }

        // apr 协议处理器边界条件处理。ARP(Address Resolution Protocol)，地址解析协议，根据 ip 地址获取物理地址的一个 TCP/IP 协议。
        if (protocolHandler.isAprRequired() && !AprLifecycleListener.isInstanceCreated()) {
            throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerNoAprListener",
                    getProtocolHandlerClassName()));
        }
        if (protocolHandler.isAprRequired() && !AprLifecycleListener.isAprAvailable()) {
            throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerNoAprLibrary",
                    getProtocolHandlerClassName()));
        }
        // 判断 apr 协议处理器是否开启 SSL
        if (AprLifecycleListener.isAprAvailable() && AprLifecycleListener.getUseOpenSSL() &&
                protocolHandler instanceof AbstractHttp11JsseProtocol) {
            AbstractHttp11JsseProtocol<?> jsseProtocolHandler =
                    (AbstractHttp11JsseProtocol<?>) protocolHandler;
            if (jsseProtocolHandler.isSSLEnabled() &&
                    jsseProtocolHandler.getSslImplementationName() == null) {
                // OpenSSL is compatible with the JSSE configuration, so use it if APR is available
                jsseProtocolHandler.setSslImplementationName(OpenSSLImplementation.class.getName());
            }
        }

        try {
            // 协议处理器执行初始化(protocolHandler 是在 server.xml 解析时创建的)
            protocolHandler.init();
        } catch (Exception e) {
            throw new LifecycleException(
                    sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }
    }
```



## Bootstrap#start 方法

