# IOC（控制反转）和DI（依赖注入）



## IOC

### 基础信息

IOC Inverse of Control （控制反转），将原本在程序中手动创建UserService对象的控制权，交给Spring，由Spring来管理（使用工厂模式）。

e.g.

`applicationContext.xml`

```xml
<!-- UserServiceImplA的创建权转交给了Spring -->
<bean id="userSerivce" class="com.example.ioc.demo.UserServiceImplA"></bean>
```

### 构成

配置文件 + 工厂 + 反射



## DI

DI Dependency Injection（依赖注入），在Spring创建这个对象的过程中（实例化过程中），将这个对象所依赖的属性注入进去。

### Srping的属性注入

对于类成员变量，注入方式有三种

- 构造方法注入
- setter方法注入
- 接口注入

Spring支持前两种

####构造方法注入

通过构造方法注入Bean的属性值或依赖的对象，它保证了Bean实例在实例化后就可以使用。

构造器注入在`<constructor-arg>`元素里声明属性。

##### e.g.

```java
public class User {
    private String name;
    private Integer age;

    public User(String name,Integer age){
        this.name = name;
        this.age = age;
    }
  	
  	@Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```



```xml
<!--applicationContext.xml-->
<!--Bean的构造方法的属性注入-->
<bean id="user" class="com.demo4.User">
	<constructor-arg name="name" value="张三" />
  <constructor-arg name="age" value="23"/>
</bean>
```



```java
@Test
public void demo1(){
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
    User user = (User)applicationContext.getBean("user");
    System.out.println(user);
}
```

#### setter方法注入

使用 setter方法注入属性，在applicationContext.xml中，通过`<property>`设置注入的属性。

##### e.g.

```java
public class Person {
    private String name;
    private Integer age;

    private Cat cat;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Cat getCat() {
        return cat;
    }

    public void setCat(Cat cat) {
        this.cat = cat;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", cat=" + cat +
                '}';
    }
}
```



```xml
<!--applicationContext.xml-->
<!--Bean的setter方法的属性注入-->
<bean id="person" class="com.imooc.ioc.demo4.Person">
    <property name="name" value="李四"/>
    <property name="age" value="32"/>
    <!--ref指引用的bean的id，当需要注入的属性是一个类的时候，可以使用这种方式-->
    <property name="cat" ref="cat"/>
</bean>

<bean id="cat" class="com.demo4.Cat">
    <property name="name" value="ketty"/>
</bean>
```



```java
@Test
public void demo2(){
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
    Person person = (Person)applicationContext.getBean("person");
    System.out.println(person);
}
```

#### p名称空间

为了简化XML文件配置，Spring从2.5开始引入一个新的p名称空间。

p:<属性名>="xxx"引入常量值。

p:<属性名>-ref="xxx"引用其它Bean对象。

#####e.g.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--此处xmlns:p="http://www.springframework.org/schema/p"为开启Spring的p名称空间所需-->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--Bean的p名称空间的属性注入==============================-->
    <bean id="person" class="com.imooc.ioc.demo4.Person" p:name="大黄" p:age="34" p:cat-ref="cat"/>

    <bean id="cat" class="com.imooc.ioc.demo4.Cat" p:name="小黄"/>
</beans>
```

#### SpEL注入

SpEL: spring expression language, spring表达式语言，对依赖注入进行简化。

语法：`#{表达式}`

`<bean id="" value=#{表达式}>`

```SpEL
// SpEL表达式语言
语法：#{}
#{'hello'}：使用字符串
#{beanId}：使用另一个bean
#{beanId.content.toUpperCase}：使用指定属性名，并使用方法
#{T(java.lang.Math).PI}：使用静态字段或方法
```

#### 复杂类型的属性注入

- 数组类型的属性注入
- List集合类型的属性注入
- Set集合类型的属性注入
- Map集合类型的属性注入
- Properties类型的属性注入

复杂类型的属性注入一般在Spring整合其它框架的时候使用。

##### e.g.

```java
public class CollectionBean {
    private String[] arrs; // 数组类型

    private List<String> list;// List集合类型

    private Set<String> set; // Set集合类型

    private Map<String,Integer> map;// Map集合类型

    private Properties properties; // 属性类型

    public String[] getArrs() {
        return arrs;
    }

    public void setArrs(String[] arrs) {
        this.arrs = arrs;
    }

    public List<String> getList() {
        return list;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    public Set<String> getSet() {
        return set;
    }

    public void setSet(Set<String> set) {
        this.set = set;
    }

    public Map<String, Integer> getMap() {
        return map;
    }

    public void setMap(Map<String, Integer> map) {
        this.map = map;
    }

    public Properties getProperties() {
        return properties;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    @Override
    public String toString() {
        return "CollectionBean{" +
                "arrs=" + Arrays.toString(arrs) +
                ", list=" + list +
                ", set=" + set +
                ", map=" + map +
                ", properties=" + properties +
                '}';
    }
}
```



```xml
<!--复杂类型的属性注入-->
<bean id="collectionBean" class="com.imooc.ioc.demo5.CollectionBean">
    <!--数组类型-->
    <property name="arrs">
        <list>
            <value>aaa</value>
            <value>bbb</value>
            <value>ccc</value>
        </list>
    </property>
    <!--List集合的属性注入-->
    <property name="list">
        <list>
            <value>111</value>
            <value>222</value>
            <value>333</value>
        </list>
    </property>
    <!--Set集合的属性注入-->
    <property name="set">
        <set>
            <value>ddd</value>
            <value>eee</value>
            <value>fff</value>
        </set>
    </property>
    <!--Map集合的属性注入-->
    <property name="map">
        <map>
            <entry key="aaa" value="111"/>
            <entry key="bbb" value="222"/>
            <entry key="ccc" value="333"/>
        </map>
    </property>
    <!--Properties的属性注入-->
    <property name="properties">
        <props>
            <prop key="username">root</prop>
            <prop key="password">1234</prop>
        </props>
    </property>
</bean>
```



## 注入的对象和new的对象的区别

注入的对象，只会调用空参构造函数，且这个对象的所有属性都是默认值，自己手动赋予的值不会被使用。 

所以在A中的C类，在B中调用A的时候，创建的A对象的属性都是默认值，所以A对象虽然有了，但是A的属性C却是null，所以在B中直接this.a.c.method()是会报null指针异常的，且是c是null的发生原因。

解决方式：C在A中添加get方法，然后B中使用a.getC()即可获得c的对象，且c的对象也是spring注入的（体现了DI 依赖注入）。

