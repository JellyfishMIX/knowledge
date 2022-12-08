# aop 总结



## 说明

1. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
2. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## aop 不生效的情况

1. 切面通过代理方式织入了方法 B。同一个类里 A 方法直接调用 B 方法，相当于 this. 调用，没有用上代理对象切面不能生效。



## 解决方案

1. 把这个类当作 bean 注入到类的属性里，然后用 bean.BMethod 的方式调用，注入进来的 bean 就是代理对象，不是真实的 bean。即自己注入自己，这是最简单的方法。
2. 还有其他方法，手动去 spring 或 aop 上下文里拿代理对象，本质还是要拿到代理对象来调用方法 B。参考: https://blog.csdn.net/weixin_38370441/article/details/113475744



## 三种 aop 方式受方法修饰符影响

1. jdk 是代理接口，private 方法没在接口里声明肯定切不到。
2. cglib 是生成子类，private 方法子类无法继承，也切不到。
3. aspectJ 编译期织入，直接改代码，private, static, final 都能织入。