# sentinel 核心概念 -- 规则 Rule



## 资源和规则

1. 资源：可以是一个方法、一个接口或一段代码。以一个 HTTP 接口为例，假设我们为 `/a/b/c`这个接口配置了限流 QPS 为 100 的限制，那么 `/a/b/c` 就被称为资源。资源是被 sentinel 保护和管理的对象，sentinel 通过对资源的调用进行限流、熔断降级等控制。
2. 规则：用来定义资源应该遵循的约束条件。在上述示例中，为 `/a/b/c` 接口配置的 QPS 为 100 的限制就是一个规则。sentinel 支持多种规则类型，如流控规则、熔断降级规则以及其他自定义规则。规则可以通过多种方式配置，例如 API、文件、配置中心等。
3. 在 sentinel 中，资源和规则是相互关联的。sentinel 会根据配置的规则，对资源的调用进行相应的控制，从而保证系统的稳定性和高可用。
4. 了解规则概念前请先了解资源概念。



## Rule 接口

```java
public interface Rule {

    /**
     * Get target resource of this rule.
     * 获取此条规则对应的资源名
     *
     * @return target resource of this rule
     */
    String getResource();

}
```

### Rule 的子类实现

![image-20231211193101639](https://image-hosting.jellyfishmix.com/20231211193101.png)



## Rule 规则变更时的通知机制 -- 观察者模式

### 观察者模式简述

1. 观察者模式的定义(GoF《设计模式》): 在对象之间定义一个一对多的依赖，当一个对象状态改变的时候，所有依赖的对象都会自动收到通知。
2. 观察者模式核心模型有两个，这里简单提一下:
   1. 一个是 Watcher 用来感知消息来触发自定义的行为。通常每个 watcher 与需要感知消息的消费方关联(例如客户端)。Watcher 的别名有很多，例如 Listener。
   2. 一个是 Subject 用来管理 watcher，例如收集指定 key 对应的 watcher 有哪些，生产方某些行为生产消息后，通过 Subject 来获取 key 对应的 watcher，进而触发 watcher 的感知。Subject 可以简化称为 WatcherManager。



// todo 具体观察者模式的应用有时间了再补坑...