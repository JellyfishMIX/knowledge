# Spring Bean



## 实例化Bean的三种方式

- 使用类构造器实例化（默认无参）[常用]
- 使用静态工厂方法实例化（简单工厂模式）
- 使用实例工厂方法实例化（工厂方法模式）



## Bean的配置

- id和name
  - 一般情况下，装配一个Bean时，通过指定一个id属性作为Bean的名称。
  - id属性在IOC容器中必须是唯一的。
  - 如果Bean的名称中含有特殊字符，则必须使用name属性。
  - 通常使用id或name都行，没有其它区别，习惯使用id。

- class
  - class用于设置一个类的完全路径名称，主要作用是IOC容器生成类的实例。

- scope
  - 指定Bean的作用域。



## Bean的作用域

| 作用域         | 描述                                                         |
| :------------- | :----------------------------------------------------------- |
| singleton      | 该作用域将 bean 的定义的限制在每一个 Spring IoC 容器中的一个单一实例（单例）[默认]。 |
| prototype      | 该作用域将单一 bean 的定义限制在任意数量的对象实例。（多例） |
| request        | 该作用域将 bean 的定义限制为 HTTP 请求。只在 web-aware Spring ApplicationContext 的上下文中有效。 |
| session        | 该作用域将 bean 的定义限制为 HTTP 会话。 只在web-aware Spring ApplicationContext的上下文中有效。 |
| global-session | 该作用域将 bean 的定义限制为全局 HTTP 会话。只在 web-aware Spring ApplicationContext 的上下文中有效。 |



## Bean的生命周期

Spring初始化Bean或销毁Bean时，有时需要做一些处理工作，因此Spring可以在创建和销毁Bean的时候调用Bean的两个生命周期方法。

```xml
<bean id="xxx" class="...Yoo" init-method="init" destory-method="destory">
```

init 和 destory两个方法可以自定义名称，方法需要存在...Yoo这个class中

- init-method: 当Bean被载入到容器时调用。
- destory-method: 当Bean从容器中销毁时调用（仅在`scope=singleton`时有效，因为在多例模式中，Spring不知道要销毁哪个实例）。



### 完整的生命周期

<img src="https://image-hosting.jellyfishmix.com/20200617122306.png" alt="截屏2020-06-17下午12.22.42" style="zoom:30%;" />

1. instantiate 对象实例化

2. populate properties 封装属性

3. 如果Bean实现了BeanNameAware，则执行setBeanName

   ```java
   @Override
   public void setBeanName(String name) {
     System.out.println("第三步：设置Bean的名称" + name);
   }
   ```

4. 如果Bean实现了BeanFactoryAware 或者 ApplicationContextAware设置工厂setBeanFactory或者上下文对象setApplicationContext

   ```java
   @Override
   public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
     System.out.println("了解工厂信息");
   }
   ```

5. 如果存在类实现了BeanPostProcesser（后处理Bean），执行postProcessBeforeInitialization

6. 如果Bean实现了InitializingBean，执行afterPropertiesSet

   ```java
   @Override
   public void afterPropertiesSet() throws Exception {
       System.out.println("属性设置后");
   }
   ```

7. 调用`<bean init-method="customerInit">`指定自定义初始化方法init

8. 如果存在类实现了BeanPostProcesser（后处理Bean），执行postProcessAfterInitialization

9. 执行业务方法（存在类中本身被调用的方法）
10. 如果Bean实现了DisposableBean，执行destory（Spring自带的一个销毁方法）
11. 调用`<bean destory-method="customerDestory">`指定自定义销毁方法



其中，关键的是5. 和 8.，可以对我们的类进行增强。

```java
public class MyBeanProcessor implements BeanPostProcesser {
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    System.out.println("第五步，BeanPostProcesser的初始化前方法");
  }
  
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    System.out.println("第八步，BeanPostProcesser的初始化后方法");
  }
}
```

步骤5. 和8. 与AOP有关联。

AOP增强：

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        //System.out.println("第五步：初始化前方法...");
        if("userDao".equals(beanName)){
            Object proxy = Proxy.newProxyInstance(bean.getClass().getClassLoader(), bean.getClass().getInterfaces(), new InvocationHandler() {
              	// 内部类
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    if("save".equals(method.getName())){
                        System.out.println("增强：权限校验===================");
                        return method.invoke(bean,args);
                    }
                    return method.invoke(bean,args);
                }
            });
            return proxy;
        }else{
            return bean;
        }
    }

    @Override
    public Object postProcessAfterInitialization(final Object bean, String beanName) throws BeansException {
        //System.out.println("第八步：初始化后方法...");
        return bean;
    }
}
```

