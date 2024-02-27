# spring-ioc 依赖注册 -- xml 配置与注解配置,组件扫描机制



## 说明

1. 本文基于 jdk 8, spring-framework 5.2.x 编写。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## 依赖注册

依赖注册指告知 ApplicationContext 想向其添加的 bean 信息，即向 ApplicationContext 注册 BeanDefinition。

一般依赖注册有两种方式，一种是手动使用 xml 或注解的方式手动配置 bean 信息。一种是将类声明为组件，借助组件扫描机制注册。



## xml 与注解配置举例

以下 xml 配置与注解配置都可以注册一个 beanName 为 person 的 bean

xml 配置

```xml
<bean id="person" class="com.demo.springexplore.Person">
    <property name="name" value="xiaoming"/>
    <property name="age" value="18"/>
</bean>
```

注解配置

```java
@Configuration
public class QuickstartConfiguration {
    @Bean("person")
    public Person person() {
        Person person = new Person();
        person.setName("xiaoming");
        person.setAge(18);
        return person;
    }
}
```

tips: 若使用 xml 配置了一个 beanName 为 person 的 bean，同时在配置类中也声明了相同 beanName 的 bean，则使用 xml 配置的 bean 优先级更高。



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



## 组件扫描机制

@ComponentScan 注解会扫描指定 package 路径下所有类，找到 @Component 及其派生类 @Controller,@Service,@Repository,@Configuration 等标注的类，向 ApplicationContext 注册 bean 信息。

```java
@Configuration
@ComponentScan("com.demo.springexplore")
public class ComponentScanConfiguration {
    
}
```

如果不写 @ComponentScan，也是可以做到组件扫描的，AnnotationConfigApplicationContext 的构造方法，支持直接传入扫描目标 package 路径。

```java
ApplicationContext ctx = new AnnotationConfigApplicationContext("com.demo.springexplore");
```



## 检查 ApplicationContext 中的 bean 信息(BeanDefinition)

```java
    public static void main(String[] args) throws Exception {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(ComponentScanConfiguration.class);
        String[] beanNames = ctx.getBeanDefinitionNames();
        Stream.of(beanNames).forEach(System.out::println);
    }
```



## 结束语

本文是一篇使用简述，具体 spring-ioc 依赖注册 -- xml 配置与注解配置,组件扫描机制的原理，需要了解 spring-ioc BeanDefinition 的解析与注册。
