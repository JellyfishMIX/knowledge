# java SPI 机制与 dubbo 对 SPI 机制的扩展



## 说明

1. 本文基于 jdk 8, dubbo 2.7.18 写作。
2. @author [JellyfishMIX - github](https://github.com/JellyfishMIX) / [blog.jellyfishmix.com](http://blog.jellyfishmix.com)
3. LICENSE [GPL-2.0](https://github.com/JellyfishMIX/GPL-2.0)



## java SPI 机制

### 介绍

SPI，全称为 Service Provider Interface，是一种服务发现机制，在 classpath 路径下的 META-INF/service 路径查找以接口的全限定名命名的文件，文件内容是接口实现类的全限定名，自动加载实现类。

简单来说是一种动态替换发现机制，接口运行时才发现具体的实现类，只需要在运行前添加一个实现即可，并且将实现描述给 jdk 即可，这里的描述便是上面提到的 META-INF/service 下的配置。也可以随时对该描述进行修改，完成具体实现的替换。

### 通过 demo 演示说明什么是 SPI 机制

介绍的表述比较官方，读完不明白。通过 demo 演示更生动。

我们写一个接口 ClearStrategy，抽象出一个 doClear 行为。这个接口即 Service Provider Interface。

```java
public interface ClearStrategy {
    void doClear();
}
```

我们提供三个实现类，这三个实现类可以理解为第三方 jar 中对 SPI 接口的实现。

```java
public class AClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 A 清理策略");
    }
}

public class BClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 B 清理策略");
    }
}

public class CClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 C 清理策略");
    }
}
```

然后我们在 resources 目录下的 META-INF/service 路径下，创建一个文件名为 jdk.spi.demo.ClearStrategy。这个目录/文件在编译后会被放在 classpath 目录下。

这个文件名和我们刚才编写的 SPI 接口同名。文件内容是各实现类的全限定名：

```
jdk.spi.demo.impl.AClearStrategy
jdk.spi.demo.impl.BClearStrategy
jdk.spi.demo.impl.CClearStrategy
```

一个测试类使用 SPI 机制加载实现类实例。

```java
public class JavaSpiLoaderTest {
    public static void main(String[] args) {
        // 这里会一次性实例化指定 SPI 接口的所有实现
        ServiceLoader<ClearStrategy> serviceLoader = ServiceLoader.load(ClearStrategy.class);
        for (ClearStrategy clearStrategy : serviceLoader) {
            clearStrategy.doClear();
        }
    }
}
```

输出结果如下，执行了每个实例重写的方法：

```
执行 A 清理策略
执行 B 清理策略
执行 C 清理策略
```

### 应用

java 提供了很多 SPI，允许第三方为这些接口提供实现。常见的 SPI 有 JDBC, JNDI 等。这些SPI接口是由Java来提供的，而SPI的实现则是作为依赖的第三方jar加载进 classpath 中。

例如：java 提供的 java.sql.Driver 和 mysql 实现的 com.mysql.cj.jdbc.Driver，并且在 META-INF/service 下存在该配置：com.mysql.cj.jdbc.Driver。



## dubbo 对 SPI 机制的扩展

### 介绍

dubbo 对 java 的 SPI 机制进行了扩展。支持给 SPI 的实现类命名，且可以单独实例化指定的 extension 实例，不需要一次性把所有实现类都实例化。

### dubbo SPI demo

我们把刚才的 SPI 接口 ClearStrategy，加上 @SPI 注解，表示这是一个 dubbo SPI。

```java
@SPI
public interface ClearStrategy {
    void doClear();
}
```

提供的三个实现类不用改动，这三个实现类是 dubbo SPI 接口的的实现类，dubbo 中称 dubbo SPI 接口的实现类为 extension。

```java
public class AClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 A 清理策略");
    }
}

public class BClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 B 清理策略");
    }
}

public class CClearStrategy implements ClearStrategy {
    @Override
    public void doClear() {
        System.out.println("执行 C 清理策略");
    }
}
```

META-INF/services 路径中，以 SPI 接口全限定名命名的配置文件，文件内容在实现类的全限定名前面加上 name，表示此 extension 的 name。

```
aStrategy=dubbo.extension.demo.impl.AClearStrategy
bStrategy=dubbo.extension.demo.impl.BClearStrategy
cStrategy=dubbo.extension.demo.impl.CClearStrategy
```

一个测试类使用 dubbo SPI 机制加载 extension。

```java
public class DubboSpiLoaderTest {
    public static void main(String[] args) {
        // 获取 dubbo SPI 接口 ClearStrategy 对应的 ExtensionLoader
        ExtensionLoader<ClearStrategy> strategyExtensionLoader = ExtensionLoader.getExtensionLoader(ClearStrategy.class);
        // 通过 extension name，获取 dubbo SPI 接口的一个实现类(extension)
        ClearStrategy clearStrategy = strategyExtensionLoader.getExtension("bStrategy");
        clearStrategy.doClear();
    }
}
```

输出结果如下，执行了获取的 extension "bStrategy" 重写的方法：

```
执行 B 清理策略
```

1. 获取 dubbo SPI 接口 ClearStrategy 对应的 ExtensionLoader
2. 通过 extension name，获取 dubbo SPI 接口的一个实现类(extension)

由上可以看出，dubbo 对 java 的 SPI 机制进行了扩展。支持给 SPI 的实现类命名，且可以单独实例化指定的 extension 实例，不需要一次性把所有实现类都实例化。