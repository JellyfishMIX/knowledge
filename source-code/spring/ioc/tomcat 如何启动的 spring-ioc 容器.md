# tomcat 如何启动的 spring-ioc 容器



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 给 tomcat 指定的上下文加载路径 web.xml

1. 查看 tomcat 中 /conf/context.xml 文件。
2. 本文中我们不关注 tomcat 的机制，只需要知道在这个 xml 中，给 tomcat 指定了上下文加载路径: WEB-INF/web.xml。

```xml
<Context>
    <!-- Default set of monitored resources -->
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
</Context>
```

一个比较常见的 web.xml 配置，可以看到配置了一个 listener 实现, ContextLoaderListener。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xsi:schemaLocation="http://java.sun.com/xml/ns/javaee  http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
		version="2.5">
	<display-name>spring-action</display-name>
	
	<!-- 定义 contextConfigLocation 的位置 -->
	<context-param>
		<param-name>contextConfigLocation</param-name>
		<param-value>classpath*:/spring/my-application-context.xml</param-value>
	</context-param>
	
	<!-- 上下文启动的监听器 -->
	<listener>
		<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	</listener>
</web-app>
```



## ServletContextListener

javax.servlet.ServletContextListener

ServletContextListener 来自于 javax.servlet 包，是 tomcat 与 spring 的衔接接口，可以监听 tomcat servletContext 相关事件，拥有操作 servlet 上下文的能力。

1. 此接口有两个方法，contextInitialized 初始化上下文，contextDestroyed 销毁上下文。

```java
package javax.servlet;

import java.util.EventListener;

/**
 * 是 tomcat 与 spring 的衔接接口，可以监听 tomcat servletContext 相关事件，拥有操作 servlet 上下文的能力。
 */
public interface ServletContextListener extends EventListener {
    /**
     * 初始化上下文
     */
    void contextInitialized(ServletContextEvent var1);
	
    /**
     * 销毁上下文
     */
    void contextDestroyed(ServletContextEvent var1);
}
```



## ContextLoaderListener

org.springframework.web.context.ContextLoaderListener

1. 实现了 ServletContextListener 接口，是 tomcat 与 spring 的衔接的实现，可以监听 tomcat servletContext 相关事件，拥有操作 servlet 上下文的能力。
2. tomcat 的 ServletContextEvent 会被 ServletContextListener#contextInitialized 方法所处理，在本文中执行的是具体实现 ContextLoaderListener#contextInitialized 方法，初始化 web 应用上下文。执行了 ContextLoader#initWebApplicationContext 方法。

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {

	public ContextLoaderListener() {
	}

	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}

	/**
	 * Initialize the root web application context.
	 *
	 * 初始化 web 应用上下文
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}

	/**
	 * Close the root web application context.
	 *
	 * 销毁 web 应用上下文
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```



## ContextLoader#initWebApplicationContext 方法

org.springframework.web.context.ContextLoader#initWebApplicationContext

初始化 web 应用上下文

1. 创建当前应用上下文。
2. 重点关注这处调用，ContextLoader#configureAndRefreshWebApplicationContext 方法，配置和刷新当前容器。

```java
	/**
	 * Initialize Spring's web application context for the given servlet context,
	 * using the application context provided at construction time, or creating a new one
	 * according to the "{@link #CONTEXT_CLASS_PARAM contextClass}" and
	 * "{@link #CONFIG_LOCATION_PARAM contextConfigLocation}" context-params.
	 *
	 * 初始化 web 应用上下文
	 *
	 * @param servletContext current servlet context
	 * @return the new WebApplicationContext
	 * @see #ContextLoader(WebApplicationContext)
	 * @see #CONTEXT_CLASS_PARAM
	 * @see #CONFIG_LOCATION_PARAM
	 */
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
			// ...
        
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
        	// 创建当前应用上下文
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					// 配置和刷新当前容器
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
			
            // ...
	}
```



## ContextLoader#configureAndRefreshWebApplicationContext 方法

org.springframework.web.context.ContextLoader#configureAndRefreshWebApplicationContext

配置和刷新当前容器。

1. 重点是调用了 ConfigurableWebApplicationContext#refresh 方法，进行 spring web 应用上下文的刷新。具体实现是 AbstractApplicationContext#refresh, 这里就和 spring-ioc 经典机制串起来了。
2. 其实 AbstractApplicationContext#refresh 方法也是被具体的应用上下文实现通过继承的方式调用的，具体的应用上下文是什么可以看 ContextLoader#initWebApplicationContext 方法中调用的 ContextLoader#createWebApplicationContext 方法。
3. 默认使用的应用上下文是 XmlWebApplicationContext

```java
	/**
	 * 配置和刷新当前容器
	 */
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}

		wac.setServletContext(sc);
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
		/*
		 * 调用 ConfigurableWebApplicationContext#refresh 方法，具体实现是 AbstractApplicationContext#refresh
		 * 其实 AbstractApplicationContext#refresh 方法也是被具体的应用上下文实现通过继承的方式调用的，
		 * 具体的应用上下文是什么可以看 ContextLoader#initWebApplicationContext 方法中调用的 ContextLoader#createWebApplicationContext 方法。
		 * 默认使用的应用上下文是 XmlWebApplicationContext
		 */
		wac.refresh();
	}
```



## ContextLoader#createWebApplicationContext 方法

org.springframework.web.context.ContextLoader#createWebApplicationContext

创建指定的应用上下文

1. 获取应用上下文的 clazz
2. 创建应用上下文实例。

```java
	/**
	 * Instantiate the root WebApplicationContext for this loader, either the
	 * default context class or a custom context class if specified.
	 * <p>This implementation expects custom contexts to implement the
	 * {@link ConfigurableWebApplicationContext} interface.
	 * Can be overridden in subclasses.
	 * <p>In addition, {@link #customizeContext} gets called prior to refreshing the
	 * context, allowing subclasses to perform custom modifications to the context.
	 *
	 * 创建指定的应用上下文
	 *
	 * @param sc current servlet context
	 * @return the root WebApplicationContext
	 * @see ConfigurableWebApplicationContext
	 */
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		// 获取应用上下文的 clazz
		Class<?> contextClass = determineContextClass(sc);
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		// 创建应用上下文实例
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```



## ContextLoader#determineContextClass 方法

org.springframework.web.context.ContextLoader#determineContextClass

获取应用上下文的 clazz

1. 尝试获取 servletContext 中指定的应用上下文名称。
2. 如果 servletContext 中指定了，则获取指定的应用上下文 clazz。指定方式和 web.xml 中设置的两个参数有关 contextClass, contextConfigLocation
3. 如果 servletContext 中未指定，则获取默认的应用上下文 clazz。默认的应用上下文来源，在 defaultStrategies 中获得的，要看 defaultStrategies 初始化的位置，在 ContextLoader 的 static 代码块中。

```java
	/**
	 * Return the WebApplicationContext implementation class to use, either the
	 * default XmlWebApplicationContext or a custom context class if specified.
	 *
	 * 获取应用上下文的 clazz
	 *
	 * @param servletContext current servlet context
	 * @return the WebApplicationContext implementation class to use
	 * @see #CONTEXT_CLASS_PARAM
	 * @see org.springframework.web.context.support.XmlWebApplicationContext
	 */
	protected Class<?> determineContextClass(ServletContext servletContext) {
		// 尝试获取 servletContext 中指定的应用上下文名称
		String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
		/*
		 * 如果 servletContext 中指定了，则获取指定的应用上下文 clazz。指定方式和 web.xml 中设置的两个参数有关 contextClass, contextConfigLocation
		 * 指定的应用上下文来源，是在 web.xml 中设置的。contextClass 和 contextConfigLocation
		 */
		if (contextClassName != null) {
			try {
				return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load custom context class [" + contextClassName + "]", ex);
			}
		}
		/*
		 * 如果 servletContext 中未指定，则获取默认的应用上下文 clazz
		 * 默认的应用上下文来源，在 defaultStrategies 中获得的，要看 defaultStrategies 初始化的位置，在 ContextLoader 的 static 代码块中
		 */
		else {
			contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
			try {
				return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load default context class [" + contextClassName + "]", ex);
			}
		}
	}
```

### 默认的应用上下文来源

默认的应用上下文来源，在 defaultStrategies 中获得的，要看 defaultStrategies 初始化的位置，在 ContextLoader 的 static 代码块中。

1. static 代码块中可以看到默认的应用上下文来源是 ContextLoader.properties
2. ContextLoader.properties 中配置的默认应用上下文是 XmlWebApplicationContext

```java
	/**
	 * Name of the class path resource (relative to the ContextLoader class)
	 * that defines ContextLoader's default strategy names.
	 */
	private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";


	private static final Properties defaultStrategies;

	static {
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
		}
	}
```

org/springframework/web/context/ContextLoader.properties

```properties
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```



## contextClass, contextConfigLocation 参数

指定使用的应用上下文，指定方式和 web.xml 中设置的两个参数有关 contextClass, contextConfigLocation

### contextClass

指定应用上下文的 class，不同的 contextClass，使用的配置文件类型不同(例如 xml 或 java 配置类):

1. org.springframework.web.context.support.XmlWebApplicationContext，使用的配置文件类型是 xml，这是传统的 spring 应用使用的应用上下文。

2. org.springframework.web.context.support.AnnotationConfigWebApplicationContext，使用的配置文件类型是 java 配置类，这是 spring boot 应用使用的应用上下文。

### contextConfigLocation

用来指定配置文件所在的位置:

1. 如果 contextClass 为 XmlWebApplicationContext, 指定的就是 xml 配置文件所在的位置，如果 contextClass 为 AnnotationConfigWebApplicationContext, 指定的就是 java 配置类所在的位置。如果没有指定 contextConfigLocation 的话，默认值为 "/WEB-INF/applicationContext.xml "
1. contextConfigLocation 可以指定多个配置文件，如果多个配置文件都定义了同一个 bean，则最后一个定义的 bean 会生效，最后多个的配置文件将会整合成一个新的配置文件。

