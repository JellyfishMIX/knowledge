# springframe 与 springboot 特点概述



## springboot

1. springboot 最关键的是类 SPI 机制，按照规范会读取 spring.factories 文件，获取指定 clazz 配置的的实现类全限定名，并使用反射创建实现类实例。dubbo 也有这样的 SPI 设计，只不过 dubbo 模型里把实现类叫 Extension, 加载实现类的工具叫 ExtensionLoader，本质都是类 SPI 机制。见 org.springframework.boot.SpringApplication#getSpringFactoriesInstances