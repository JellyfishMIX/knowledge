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



# eureka在阿里服务器中不能获取正确的公网ip

个别服务器，比如虚拟服务器等，很多网关网卡做了很多映射或者很多虚拟网卡，容易导致eureka 客户端不能获取到正确的公网ip地址，容易直接获取局域网IP，导致服务注册不到eureka服务端。

这种情况，把相应服务的公网ip 在配置文件中写固定。

或在eureka中使用域名来指定服务。



## eureka 配置文件

```yaml
spring:
  application:
    name: coupon-eureka

server:
  port: 8000

eureka:
  instance:
    hostname: localhost
  client:
    # 标识是否从 Eureka Server 获取注册信息, 默认是 true. 如果这是一个单节点的 Eureka Server
    # 不需要同步其他节点的数据, 设置为 false
    fetch-registry: false
    # 是否将自己注册到 Eureka Server, 默认是 true. 由于当前应用是单节点的 Eureka Server
    # 需要设置为 false
    register-with-eureka: false
    # 设置 Eureka Server 所在的地址, 查询服务和注册服务都需要依赖这个地址
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```



##Citation/Reference

[填坑：Eureka java.net.UnknownHostException: xxxxxxxx - catoop - CSDN](https://blog.csdn.net/catoop/article/details/100989397)