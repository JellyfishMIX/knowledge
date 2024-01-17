# dubbo 服务引用源码分析



dubbo 有两种引用方式: 第一种是服务直连，第二种是基于注册中心进行引用。

服务的引用分为饿汉式和懒汉式，饿汉即调用ReferenceBean的afterPropertiesSet方法时引用；懒汉即ReferenceBean对应的服务被注入到其他类时引用，也就是用到了这个服务才会引用。Dubbo默认是懒汉式引用。

DubboNamespaceHandler 中 init 方法可以看到服务引用的入口 ReferenceBean 

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```



## ReferenceBean#getObject 方法

com.alibaba.dubbo.config.spring.ReferenceBean#getObject

获取服务引用对象。

```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
    // reference to interface impl
    private T ref;
    
	/**
	 * com.alibaba.dubbo.config.spring.ReferenceBean#getObject
	 */
	public Object getObject() throws Exception {
        return get();
    }
	
	/**
	 * com.alibaba.dubbo.config.ReferenceConfig#get
	 */
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            init();
        }
        return ref;
    }
}
```

ReferenceBean 中持有服务引用对象的引用 ref。



## ReferenceConfig#init 方法

com.alibaba.dubbo.config.ReferenceConfig#init

在创建服务引用 refer 前，校验接口信息，获取各种配置信息。

```java
private void init() {
        // 避免重复初始化
        if (initialized) {
            return;
        }
        initialized = true;
        // 检查接口名是否合法
        if (interfaceName == null || interfaceName.length() == 0) {
            throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
        }
        // get consumer's global configuration
    	// 获取消费者全局配置
        checkDefault();
        appendProperties(this);
        if (getGeneric() == null && getConsumer() != null) {
            setGeneric(getConsumer().getGeneric());
        }
    	// ...
        // 从系统变量中获取与接口名对应的属性值
        String resolve = System.getProperty(interfaceName);
        String resolveFile = null;
        // 如果没有获取到，就去加载
        if (resolve == null || resolve.length() == 0) {
            // 从系统变量中获取获取解析文件路径
            resolveFile = System.getProperty("dubbo.resolve.file");
            if (resolveFile == null || resolveFile.length() == 0) {
                File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
                if (userResolveFile.exists()) {
                    // 获取绝对路径
                    resolveFile = userResolveFile.getAbsolutePath();
                }
            }
            if (resolveFile != null && resolveFile.length() > 0) {
                Properties properties = new Properties();
                FileInputStream fis = null;
                try {
                    fis = new FileInputStream(new File(resolveFile));
                    // 从文件中加载配置
                    properties.load(fis);
                } catch (IOException e) {
                    throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
                } finally {
                    // ...
                }
                // 获取与接口对应的配置
                resolve = properties.getProperty(interfaceName);
            }
        }
    	// ...
        checkApplication();
        checkStubAndMock(interfaceClass);
        // 添加 side, 协议版本, 时间戳, 进程号等信息到 map 中
        Map<String, String> map = new HashMap<String, String>();
        Map<Object, Object> attributes = new HashMap<Object, Object>();
        map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
        map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
        map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
        if (ConfigUtils.getPid() > 0) {
            map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
        }
        // 非泛化服务
        if (!isGeneric()) {
            // 获取版本
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put("revision", revision);
            }
            // 获取方法列表，添加到 map 中
            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("NO method found in service interface " + interfaceClass.getName());
                map.put("methods", Constants.ANY_VALUE);
            } else {
                map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
            }
        }
        // 添加前面获取的各种信息到 map 中
        map.put(Constants.INTERFACE_KEY, interfaceName);
        appendParameters(map, application);
        appendParameters(map, module);
        appendParameters(map, consumer, Constants.DEFAULT_KEY);
        appendParameters(map, this);
        String prifix = StringUtils.getServiceKey(map);
        if (methods != null && methods.size() > 0) {
            for (MethodConfig method : methods) {
                // ...
                appendAttributes(attributes, method, prifix + "." + method.getName());
                checkAndConvertImplicitConfig(method, map, attributes);
            }
        }
        // 获取消费者 ip 地址
        String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
        if (hostToRegistry == null || hostToRegistry.length() == 0) {
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

        //attributes are stored by system context.
        // 存储属性值到系统上下文中，即前面获取的各种属性值
        StaticContext.getSystemContext().putAll(attributes);
        // 创建代理对象
        ref = createProxy(map);
        ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
        ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
    }
```

