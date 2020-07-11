# SpringCloud概谈



## The Bootstrap Application Context（引导上下文）

一个spring cloud应用会创建一个“bootstrap”context，它是主应用的parent context。它负责加载外部资源的配置属性并且解释本地外部配置文件中的属性。

这里有两个context，一个是spring boot的main context，另一个是spring cloud的bootstrap context，这两个context共享同一个环境，也就是说他们共享spring项目的外部配置属性。

默认情况下，bootstrap属性（并非是bootstrap.properties而是那些在bootstrap阶段加载的属性）是以高优先级的方式添加的，所以无法被本地配置覆盖。

bootstrap context和main context使用不同的方法定位外部配置，你可以使用bootstrap.yml来替代application.yml来为bootstrap context添加配置，这样就可以区分开bootstrap context和main context。

可以通过在系统属性中设置`spring.cloud.bootstrap.enabled=false`bootstrap程序。



## 引用/参考

[MMMHUHU的手记 - 慕课网](https://www.imooc.com/article/76450)
