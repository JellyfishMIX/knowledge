# Eureka



## Q&A

### 1.

#### 问题描述

1. 开发电脑上的 eureka-server 一个，服务A和服务B。使用服务A调用通过 RestTemplate 使用服务名称调用服务B，结果OK。
2. 然后将eureka 服务通过docker的方式部署到服务器A上，服务A也部署到服务器A上，服务B部署到服务器B上（三个docker容器顺利启动）
3. 测试第1步，服务A无法正常调用服务B，出现异常 `java.net.UnknownHostException: 936529518a25`

#### 解决问题

进行如下配置：

```
eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
# 下面这行是重点
eureka.instance.prefer-ip-address=true
1234
```

#### 配置解释

关于eureka 如上个配置点解释如下：

1. `eureka.client.fetch-registry`
   解释：是否检索服务（获取eureka服务列表）

2. `eureka.client.register-with-eureka`
   解释：是否向服务注册中心注册自己（如果仅作为调用者，不提供服务，可以为false）

3. `eureka.instance.prefer-ip-address`
   解释：将IP注册到eureka中，如果为false**默认注册主机名**

其中**第3点为重点**。

PS：当 `eureka.instance.ip-address` 和 `eureka.instance.prefer-ip-address` 都配置时，优先前者！



##Citation/Reference

[填坑：Eureka java.net.UnknownHostException: xxxxxxxx - catoop - CSDN](https://blog.csdn.net/catoop/article/details/100989397)