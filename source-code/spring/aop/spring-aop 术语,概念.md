# spring-aop 术语,概念



## aspectj 的术语和概念

spring 早期有一套自己的 aop 概念和术语，但没有 aspectj 流行，因此 spring 融合使用了 aspectj 的术语和概念。

1. pointcut
2. joinpoint
3. advice
4. introduction
5. @Before,@After,@Around,@...



##静态代理

以 aspectj 为代表，在编译期对生成的类的字节码做修改。

优点:

1. 节省运行时创建代理的开销。

缺点:

1. 作用在编译期，运行时生成的 class 文件的字节码无法生效。而 java 一大特性是运行时可以生成类，因此 aspectj 的适用范围较窄，目前已很少使用 aspectj 静态代理。



## 动态代理

### jdk 动态代理

指通过 InvokeHandler 传入 invoke 方法，在 invokeHandler 中织入 advice 的方式。

优点:

1. 通过子类的方式不用修改字节码，复杂度低。

缺点:

1. 对于 private 方法无法进行织入，对于未实现接口的子类，无法进行织入。

### cglib 动态代理

通过继承的方式进行代理。

优点:

1. 在运行时创建 target bean 的子类来进行 advice 的织入，可用于 target bean 没有实现接口的情况。

缺点:

1. 因为通过创建子类做 advice 的织入，因此 target bean 的 final, private 方法无法织入。