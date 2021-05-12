# 一、事务简单介绍

事务指逻辑上的一组操作，组成这组操作的各个单元，要不全部成功，要不全部不成功。

## 1.1 事务基本要素

- 原子性(Atomicity): 事务开始后所有操作，要么全部做完，要么全部不做，不可能停滞在中间环节。事务执行过程中出错，会回滚到事务开始前的状态，所有的操作就像没有发生一样。也就是说事务是一个不可分割的整体，就像化学中学过的原子，是物质构成的基本单位。
- 一致性(Consistency): 事务开始前和结束后，数据库的完整性约束没有被破坏。比如A向B转账，不可能A扣了钱，B却没收到。
- 隔离性(Isolation): 同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。比如A正在从一张银行卡中取钱，在A取钱的过程结束前，B不能向这张卡转账。
- 持久性(Durability): 事务完成后，事务对数据库的所有更新将被保存到数据库，不能回滚。

## 1.2 Spring事务属性

> Spring事务属性对应TransactionDefinition类里面的各个方法。TransactionDefinition类方法如下所示:



```csharp
public interface TransactionDefinition {

    /**
     * 返回事务传播行为
     */
    int getPropagationBehavior();

    /**
     * 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
     */
    int getIsolationLevel();

    /**
     * 事务超时时间，事务必须在多少秒之内完成
     */
    int getTimeout();

    /**
     * 事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的
     */
    boolean isReadOnly();

    /**
     * 事务名字
     */
    @Nullable
    String getName();
}
```

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。事务属性包含了5个方面：传播行为、隔离规则、回滚规则、事务超时、是否只读。

> 事务的产生需要依赖这些事务属性。包括我们下面要讲到的@Transactional注解的属性其实就是在设置这些值。

### 1.2.1 传播行为

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运行，也可能开启一个新事务，并在自己的事务中运行。Spring定义了七种传播行为：

| 传播行为                                        | 含义                                                         |
| ----------------------------------------------- | ------------------------------------------------------------ |
| TransactionDefinition.PROPAGATION_REQUIRED      | 如果当前没有事务，就新建一个事务，如果已经存在一个事务，则加入到这个事务中。这是最常见的选择。 |
| TransactionDefinition.PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| TransactionDefinition.PROPAGATION_MANDATORY     | 表示该方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常 |
| TransactionDefinition.PROPAGATION_REQUIRED_NEW  | 表示当前方法必须运行在它自己的事务中。一个新的事务将被启动。如果存在当前事务，在该方法执行期间，当前事务会被挂起。 |
| TransactionDefinition.PROPAGATION_NOT_SUPPORTED | 表示该方法不应该运行在事务中。如果当前存在事务，就把当前事务挂起。 |
| TransactionDefinition.PROPAGATION_NEVER         | 表示当前方法不应该运行在事务上下文中。如果当前正有一个事务在运行，则会抛出异常 |
| TransactionDefinition.PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |

### 1.2.2 隔离规则

隔离级别定义了一个事务可能受其他并发事务影响的程度。

在实际开发过程中，我们绝大部分的事务都是有并发情况。下多个事务并发运行，经常会操作相同的数据来完成各自的任务。在这种情况下可能会导致以下的问题:

- 脏读（Dirty reads）—— 事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据。
- 不可重复读（Nonrepeatable read）—— 事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
- 幻读（Phantom read）—— 系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

> 不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

咱们已经知道了在并发状态下可能产生:　脏读、不可重复读、幻读的情况。因此我们需要将事务与事务之间隔离。根据隔离的方式来避免事务并发状态下脏读、不可重复读、幻读的产生。Spring中定义了五种隔离规则:

| 隔离级别                                         | 含义                                                         | 脏读 | 不可重复读 | 幻读 |
| ------------------------------------------------ | ------------------------------------------------------------ | ---- | ---------- | ---- |
| TransactionDefinition.ISOLATION_DEFAULT          | 使用后端数据库默认的隔离级别                                 |      |            |      |
| TransactionDefinition.ISOLATION_READ_UNCOMMITTED | 允许读取尚未提交的数据变更(最低的隔离级别)                   | 是   | 是         | 是   |
| TransactionDefinition.ISOLATION_READ_COMMITTED   | 允许读取并发事务已经提交的数据                               | 否   | 是         | 是   |
| TransactionDefinition.ISOLATION_REPEATABLE_READ  | 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改 | 否   | 否         | 是   |
| TransactionDefinition.ISOLATION_SERIALIZABLE     | 最高的隔离级别，完全服从ACID的隔离级别，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的 | 否   | 否         | 否   |

> ​    ISOLATION_SERIALIZABLE 隔离规则类型在开发中很少用到。举个很简单的例子。咱们使用了ISOLATION_SERIALIZABLE规则。A,B两个事务操作同一个数据表并发过来了。A先执行。A事务这个时候会把表给锁住，B事务执行的时候直接报错。

​    **补充:**

- 事务隔离级别为ISOLATION_READ_UNCOMMITTED时，写数据只会锁住相应的行。
- 事务隔离级别为可ISOLATION_REPEATABLE_READ时，如果检索条件有索引(包括主键索引)的时候，默认加锁方式是next-key锁；如果检索条件没有索引，更新数据时会锁住整张表。一个间隙被事务加了锁，其他事务是不能在这个间隙插入记录的，这样可以防止幻读。
- 事务隔离级别为ISOLATION_SERIALIZABLE时，读写数据都会锁住整张表。
- 隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也就越大。

### 1.2.3 回滚规则

事务回滚规则定义了哪些异常会导致事务回滚而哪些不会。默认情况下，只有未检查异常(RuntimeException和Error类型的异常)会导致事务回滚。而在遇到检查型异常时不会回滚。 但是你可以声明事务在遇到特定的检查型异常时像遇到运行期异常那样回滚。同样，你还可以声明事务遇到特定的异常不回滚，即使这些异常是运行期异常。

### 1.2.4 事务超时

为了使应用程序很好地运行，事务不能运行太长的时间。因为事务可能涉及对后端数据库的锁定，也会占用数据库资源。事务超时就是事务的一个定时器，在特定时间内事务如果没有执行完毕，那么就会自动回滚，而不是一直等待其结束。

### 1.2.5 是否只读

如果在一个事务中所有关于数据库的操作都是只读的，也就是说，这些操作只读取数据库中的数据，而并不更新数据,　这个时候我们应该给该事务设置只读属性，这样可以帮助数据库引擎优化事务。提升效率。

# 二、@Transactional使用

Spring 为事务管理提供了丰富的功能支持。Spring 事务管理分为编码式和声明式的两种方式:

- 编程式事务:允许用户在代码中精确定义事务的边界。编程式事务管理使用TransactionTemplate或者直接使用底层的PlatformTransactionManager。对于编程式事务管理，spring推荐使用TransactionTemplate。
- 声明式事务: 基于AOP,有助于用户将操作与事务规则进行解耦。其本质是对方法前后进行拦截，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。声明式事务管理也有两种常用的方式，一种是在配置文件(xml)中做相关的事务规则声明，另一种是基于@Transactional注解的方式。显然基于注解的方式更简单易用，更清爽。@Transactional注解的使用也是我们本文着重要理解的部分。

> ​    显然声明式事务管理要优于编程式事务管理，这正是spring倡导的非侵入式的开发方式。声明式事务管理使业务代码不受污染，一个普通的POJO对象，只要加上注解就可以获得完全的事务支持。和编程式事务相比，声明式事务唯一不足地方是，后者的最细粒度只能作用到方法级别，无法做到像编程式事务那样可以作用到代码块级别。但是即便有这样的需求，也存在很多变通的方法，比如，可以将需要进行事务管理的代码块独立为方法等等。

## 2.1 @Transactional介绍

@Transactional注解 可以作用于接口、接口方法、类以及类方法上。当作用于类上时，该类的所有 public 方法将都具有该类型的事务属性，同时，我们也可以在方法级别使用该标注来覆盖类级别的定义。

虽然@Transactional 注解可以作用于接口、接口方法、类以及类方法上，但是 Spring 建议不要在接口或者接口方法上使用该注解，因为这只有在使用基于接口的代理时它才会生效。另外， @Transactional注解应该只被应用到 public 方法上，这是由Spring AOP的本质决定的。如果你在 protected、private 或者默认可见性的方法上使用 @Transactional 注解，这将被忽略，也不会抛出任何异常。

默认情况下，只有来自外部的方法调用才会被AOP代理捕获，也就是，类内部方法调用本类内部的其他方法并不会引起事务行为，即使被调用方法使用@Transactional注解进行修饰。

## 2.2 @Transactional注解属性

@Transactional注解里面的各个属性和咱们在上面讲的事务属性里面是一一对应的。用来设置事务的传播行为、隔离规则、回滚规则、事务超时、是否只读。



```tsx
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

    /**
     * 当在配置文件中有多个 TransactionManager , 可以用该属性指定选择哪个事务管理器。
     */
    @AliasFor("transactionManager")
    String value() default "";

    /**
     * 同上。
     */
    @AliasFor("value")
    String transactionManager() default "";

    /**
     * 事务的传播行为，默认值为 REQUIRED。
     */
    Propagation propagation() default Propagation.REQUIRED;

    /**
     * 事务的隔离规则，默认值采用 DEFAULT。
     */
    Isolation isolation() default Isolation.DEFAULT;

    /**
     * 事务超时时间。
     */
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;

    /**
     * 是否只读事务
     */
    boolean readOnly() default false;

    /**
     * 用于指定能够触发事务回滚的异常类型。
     */
    Class<? extends Throwable>[] rollbackFor() default {};

    /**
     * 同上，指定类名。
     */
    String[] rollbackForClassName() default {};

    /**
     * 用于指定不会触发事务回滚的异常类型
     */
    Class<? extends Throwable>[] noRollbackFor() default {};

    /**
     * 同上，指定类名
     */
    String[] noRollbackForClassName() default {};

}
```

### 2.2.1 value、transactionManager属性

它们两个是一样的意思。当配置了多个事务管理器时，可以使用该属性指定选择哪个事务管理器。大多数项目只需要一个事务管理器。然而，有些项目为了提高效率、或者有多个完全不同又不相干的数据源，从而使用了多个事务管理器。机智的Spring的Transactional管理已经考虑到了这一点，首先定义多个transactional manager，并为qualifier属性指定不同的值；然后在需要使用@Transactional注解的时候指定TransactionManager的qualifier属性值或者直接使用bean名称。配置和代码使用的例子：



```xml
<tx:annotation-driven/>
 
<bean id="transactionManager1" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="datasource1"></property>
    <qualifier value="datasource1Tx"/>
</bean>
 
<bean id="transactionManager2" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="datasource2"></property>
    <qualifier value="datasource2Tx"/>
</bean>
```



```java
public class TransactionalService {
 
    @Transactional("datasource1Tx")
    public void setSomethingInDatasource1() { ... }
 
    @Transactional("datasource2Tx")
    public void doSomethingInDatasource2() { ... }

}
```

### 2.2.2 propagation属性

propagation用于指定事务的传播行为，默认值为 REQUIRED。propagation有七种类型，就是我们在上文中讲到的事务属性传播行为的七种方式，如下所示:

| propagation属性 | 事务属性-传播行为                               | 含义                                                         |
| --------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| REQUIRED        | TransactionDefinition.PROPAGATION_REQUIRED      | 如果当前没有事务，就新建一个事务，如果已经存在一个事务，则加入到这个事务中。这是最常见的选择。 |
| SUPPORTS        | TransactionDefinition.PROPAGATION_SUPPORTS      | 支持当前事务，如果当前没有事务，就以非事务方式执行。         |
| MANDATORY       | TransactionDefinition.PROPAGATION_MANDATORY     | 表示该方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常。 |
| REQUIRES_NEW    | TransactionDefinition.PROPAGATION_REQUIRES_NEW  | 表示当前方法必须运行在它自己的事务中。一个新的事务将被启动。如果存在当前事务，在该方法执行期间，当前事务会被挂起。 |
| NOT_SUPPORTED   | TransactionDefinition.PROPAGATION_NOT_SUPPORTED | 表示该方法不应该运行在事务中。如果当前存在事务，就把当前事务挂起。 |
| NEVER           | TransactionDefinition.PROPAGATION_NEVER         | 表示当前方法不应该运行在事务上下文中。如果当前正有一个事务在运行，则会抛出异常。 |
| NESTED          | TransactionDefinition.PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |

### 2.2.3 isolation属性

isolation用于指定事务的隔离规则，默认值为DEFAULT。@Transactional的隔离规则和上文事务属性里面的隔离规则也是一一对应的。总共五种隔离规则，如下所示:

| @isolation属性   | 事务属性-隔离规则                                | 含义                                                         | 脏读 | 不可重复读 | 幻读 |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------------ | ---- | ---------- | ---- |
| DEFAULT          | TransactionDefinition.ISOLATION_DEFAULT          | 使用后端数据库默认的隔离级别                                 |      |            |      |
| READ_UNCOMMITTED | TransactionDefinition.ISOLATION_READ_UNCOMMITTED | 允许读取尚未提交的数据变更(最低的隔离级别)                   | 是   | 是         | 是   |
| READ_COMMITTED   | TransactionDefinition.ISOLATION_READ_COMMITTED   | 允许读取并发事务已经提交的数据                               | 否   | 是         | 是   |
| REPEATABLE_READ  | TransactionDefinition.ISOLATION_REPEATABLE_READ  | 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改 | 否   | 否         | 是   |
| SERIALIZABLE     | TransactionDefinition.ISOLATION_SERIALIZABLE     | 最高的隔离级别，完全服从ACID的隔离级别，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的 | 否   | 否         | 否   |

### 2.2.4 timeout

timeout用于设置事务的超时属性。

### 2.2.5 readOnly

readOnly用于设置事务是否只读属性。

### 2.2.6 rollbackFor、rollbackForClassName、noRollbackFor、noRollbackForClassName

rollbackFor、rollbackForClassName用于设置那些异常需要回滚；noRollbackFor、noRollbackForClassName用于设置那些异常不需要回滚。他们就是在设置事务的回滚规则。

## 2.3 @Transactional注解的使用

@Transactional注解的使用关键点在理解@Transactional注解里面各个参数的含义。这个咱们在上面已经对@Transactional注解参数的各个含义做了一个简单的介绍。接下来，咱们着重讲一讲@Transactional注解使用过程中一些注意的点。

@Transactional注解内部实现依赖于Spring AOP编程。而AOP在默认情况下，只有来自外部的方法调用才会被AOP代理捕获，也就是，类内部方法调用本类内部的其他方法并不会引起事务行为。

### 2.3.1 @Transactional 注解尽量直接加在方法上

为什么：因为@Transactional直接加在类或者接口上，@Transactional注解会对类或者接口里面所有的public方法都有效(相当于所有的public方法都加上了@Transactional注解，而且注解带的参数都是一样的)。第一影响性能，可能有些方法我不需要@Transactional注解，第二方法不同可能@Transactional注解需要配置的参数也不同，比如有一个方法只是做查询操作，那咱们可能需要配置Transactional注解的readOnly参数。所以强烈建议@Transactional注解直接添加的需要的方法上。

### 2.3.2 @Transactional 注解必须添加在public方法上，private、protected方法上是无效的

在使用@Transactional 的时候一定要记住，在private,protected方法上添加@Transactional 注解不会有任何效果。相当于没加一样。即使外部能调到protected的方法也无效。和没有添加@Transactional一样。

### 2.3.3 函数之间相互调用

关于有@Transactional的函数之间调用，会产生什么情况。这里咱们通过几个例子来说明。

#### 2.3.3.1 同一个类中函数相互调用

同一个类AClass中，有两个函数aFunction、aInnerFunction。aFunction调用aInnerFunction。而且aFunction函数会被外部调用。

> 情况0: aFunction添加了@Transactional注解，aInnerFunction函数没有添加。aInnerFunction抛异常。



```cpp
public class AClass {

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        aInnerFunction(); // 调用内部没有添加@Transactional注解的函数
    }

    private void aInnerFunction() {
        //todo: 操作数据B(做了增，删，改 操作)
        throw new RuntimeException("函数执行有异常!");
    }

}
```

结果：两个函数操作的数据都会回滚。

> 情况1：两个函数都添加了@Transactional注解。aInnerFunction抛异常。



```cpp
public class AClass {

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        aInnerFunction(); // 调用内部没有添加@Transactional注解的函数
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    private void aInnerFunction() {
        //todo: 操作数据B(做了增，删，改 操作)
        throw new RuntimeException("函数执行有异常!");
    }

}
```

结果：同第一种情况一样，两个函数对数据库操作都会回滚。因为同一个类中函数相互调用的时候，内部函数添加@Transactional注解无效。@Transactional注解只有外部调用才有效。

> 情况2: aFunction不添加注解，aInnerFunction添加注解。aInnerFunction抛异常。



```cpp
public class AClass {

    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        aInnerFunction(); // 调用内部没有添加@Transactional注解的函数
    }

    @Transactional(rollbackFor = Exception.class)
    protected void aInnerFunction() {
        //todo: 操作数据B(做了增，删，改 操作)
        throw new RuntimeException("函数执行有异常!");
    }

}
```

结果：两个函数对数据库的操作都不会回滚。因为内部函数@Transactional注解添加和没添加一样。

> 情况3：aFunction添加了@Transactional注解，aInnerFunction函数没有添加。aInnerFunction抛异常，不过在aFunction里面把异常抓出来了。



```cpp
public class AClass {

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        try {
            aInnerFunction(); // 调用内部没有添加@Transactional注解的函数
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void aInnerFunction() {
        //todo: 操作数据B(做了增，删，改 操作)
        throw new RuntimeException("函数执行有异常!");
    }

}
```

结果：两个函数里面的数据库操作都成功。事务回滚的动作发生在当有@Transactional注解函数有对应异常抛出时才会回滚。(当然了要看你添加的@Transactional注解有没有效)。

#### 2.3.3.1. 不同类中函数相互调用

两个类AClass、BClass。AClass类有aFunction、BClass类有bFunction。AClass类aFunction调用BClass类bFunction。最终会在外部调用AClass类的aFunction。

> 情况0：aFunction添加注解，bFunction不添加注解。bFunction抛异常。



```java
@Service()
public class AClass {

    private BClass bClass;

    @Autowired
    public void setbClass(BClass bClass) {
        this.bClass = bClass;
    }

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        bClass.bFunction();
    }

}

@Service()
public class BClass {

    public void bFunction() {
        //todo: 数据库操作A(增，删，该)
        throw new RuntimeException("函数执行有异常!");
    }
}
```

结果：两个函数对数据库的操作都回滚了。

> 情况1：aFunction、bFunction两个函数都添加注解，bFunction抛异常。



```java
@Service()
public class AClass {

    private BClass bClass;

    @Autowired
    public void setbClass(BClass bClass) {
        this.bClass = bClass;
    }

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        bClass.bFunction();
    }

}

@Service()
public class BClass {

    @Transactional(rollbackFor = Exception.class)
    public void bFunction() {
        //todo: 数据库操作A(增，删，该)
        throw new RuntimeException("函数执行有异常!");
    }
}
```

结果：两个函数对数据库的操作都回滚了。两个函数里面用的还是同一个事务。这种情况下，你可以认为事务rollback了两次。两个函数都有异常。

> 情况2：aFunction、bFunction两个函数都添加注解，bFunction抛异常。aFunction抓出异常。



```java
@Service()
public class AClass {

    private BClass bClass;

    @Autowired
    public void setbClass(BClass bClass) {
        this.bClass = bClass;
    }

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        try {
            bClass.bFunction();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

@Service()
public class BClass {

    @Transactional(rollbackFor = Exception.class)
    public void bFunction() {
        //todo: 数据库操作A(增，删，该)
        throw new RuntimeException("函数执行有异常!");
    }
}
```

结果：两个函数数据库操作都没成功。而且还抛异常了。org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only。看打印出来的解释也很好理解把。咱们也可以这么理解，两个函数用的是同一个事务。bFunction函数抛了异常，调了事务的rollback函数。事务被标记了只能rollback了。程序继续执行，aFunction函数里面把异常给抓出来了，这个时候aFunction函数没有抛出异常，既然你没有异常那事务就需要提交，会调事务的commit函数。而之前已经标记了事务只能rollback-only(以为是同一个事务)。直接就抛异常了，不让调了。

> 情况3：aFunction、bFunction两个函数都添加注解，bFunction抛异常。aFunction抓出异常。这里要注意bFunction函数@Transactional注解我们是有变化的，加了一个参数propagation = Propagation.REQUIRES_NEW，控制事务的传播行为。表明是一个新的事务。其实咱们情况3就是来解决情况2的问题的。



```java
@Service()
public class AClass {

    private BClass bClass;

    @Autowired
    public void setbClass(BClass bClass) {
        this.bClass = bClass;
    }

    @Transactional(rollbackFor = Exception.class)
    public void aFunction() {
        //todo: 数据库操作A(增，删，该)
        try {
            bClass.bFunction();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

@Service()
public class BClass {

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void bFunction() {
        //todo: 数据库操作A(增，删，该)
        throw new RuntimeException("函数执行有异常!");
    }
}
```

结果：bFunction函数里面的操作回滚了，aFunction里面的操作成功了。有了前面情况2的理解。这种情况也很好解释。两个函数不是同一个事务了。

------

关于@Transactional注解的使用，就说这么些。最后做几点总结：

1. 要知道@Transactional注解里面每个属性的含义。@Transactional注解属性就是来控制事务属性的。通过这些属性来生成事务。
2. 要明确我们添加的@Transactional注解会不会起作用。@Transactional注解在外部调用的函数上才有效果，内部调用的函数添加无效，要切记。这是由AOP的特性决定的。
3. 要明确事务的作用范围，有@Transactional的函数调用有@Transactional的函数的时候，进入第二个函数的时候是新的事务，还是沿用之前的事务。稍不注意就会抛UnexpectedRollbackException异常。



## 转自

作者：tuacy
链接：https://www.jianshu.com/p/befc2d73e487
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。