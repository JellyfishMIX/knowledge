# logback-spring.xml读取yml参数



## 前景提要

使用logback做日志，xml中会希望动态读取yml中的配置参数：

1. 日志的输出位置可能要根据部署的环境动态进行配置。
2. 读取yml中的日志级别。而logbackxm.xml中使用${[xxx.xxx.xxx](http://xxx.xxx.xxx/)}读取不到系统参数。



## 解决方法

### 步骤一：使用logback-spring.xml

将原先的logback.xml改成logback-spring.xml。原因是springboot先读取logback.xml，然后加载yml/properties，再加载logback-spring.xml。官网上的解释如下。
![在这里插入图片描述](https://image-hosting.jellyfishmix.com/20201130152934.png)

### 步骤二：xml中增加springProperty

yml中增加自定义参数，这个参数是自定义的，写啥都行。

```yml
my:
  log:
    path: E:/jd-gateway/logdir
```

然后xml中配置如下参数，此处注意source要和上面的yml中配置的一致：

```xml
<!-- 获取yml中的log地址 -->
<springProperty scope="context" name="logPath" source="my.log.path" defaultValue="logdir"/>
```

这样，在xml中就可以使用yml中的参数了：

```xml
<!-- 日志文件存储位置 -->
<property name="log.dir" value="${logPath}" /> 
```



## 转自

[SpringBoot中，logback.xml里面读取yml里面的参数方式 - Doubletree_lin - CSDN](https://blog.csdn.net/qq_27808011/article/details/98730608)