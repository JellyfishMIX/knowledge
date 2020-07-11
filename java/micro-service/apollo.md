# apollo



## 核心概念

![截屏2020-07-10下午7.49.19](https://image-hosting.jellyfishmix.com/20200710194935.png)

1. **application (应用)**
   - 这个很好理解，就是实际使用配置的应用，Apollo客户端在运行时需要知道当前应用是谁，从而可以去获取对应的配置。
   - 每个应用都需要有唯一的身份标识 -- appId，我们认为应用身份是跟着代码走的，所以需要在代码中配置，具体信息请参见[Java客户端使用指南](https://github.com/ctripcorp/apollo/wiki/Java客户端使用指南)。
2. **environment (环境)**
   - 配置对应的环境，Apollo客户端在运行时需要知道当前应用处于哪个环境，从而可以去获取应用的配置。
   - 我们认为环境和代码无关，同一份代码部署在不同的环境就应该能够获取到不同环境的配置。
   - 所以环境默认是通过读取机器上的配置（server.properties中的env属性）指定的，不过为了开发方便，我们也支持运行时通过System Property等指定，具体信息请参见[Java客户端使用指南](https://github.com/ctripcorp/apollo/wiki/Java客户端使用指南)。
3. **cluster (集群)**
   - 一个应用下不同实例的分组，比如典型的可以按照数据中心分，把上海机房的应用实例分为一个集群，把北京机房的应用实例分为另一个集群。
   - 对不同的cluster，同一个配置可以有不一样的值，如zookeeper地址。
   - 集群默认是通过读取机器上的配置（server.properties中的idc属性）指定的，不过也支持运行时通过System Property指定，具体信息请参见[Java客户端使用指南](https://github.com/ctripcorp/apollo/wiki/Java客户端使用指南)。
4. **namespace (命名空间)**
   - 一个应用下不同配置的分组，可以简单地把namespace类比为文件，不同类型的配置存放在不同的文件中，如数据库配置文件，RPC配置文件，应用自身的配置文件等。
   - 应用可以直接读取到公共组件的配置namespace，如DAL，RPC等。
   - 应用也可以通过继承公共组件的配置namespace来对公共组件的配置做调整，如DAL的初始数据库连接数。



## 工作原理

![img](https://image-hosting.jellyfishmix.com/20200710192354.png)



## 四个核心模块及其主要功能

1. ConfigService

   提供配置获取接口
   提供配置推送接口
   服务于Apollo客户端

2. AdminService

   提供配置管理接口
   提供配置修改发布接口
   服务于管理界面Portal

3. Client

   为应用获取配置，支持实时更新
   通过MetaServer获取ConfigService的服务列表
   使用客户端软负载SLB方式调用ConfigService

4. Portal

   配置管理界面
   通过MetaServer获取AdminService的服务列表
   使用客户端软负载SLB方式调用AdminService



## 三个辅助服务发现模块

1. Eureka

   用于服务发现和注册
   Config/AdminService注册实例并定期报心跳
   和ConfigService住在一起部署

2. MetaServer

   Portal通过域名访问MetaServer获取AdminService的地址列表
   Client通过域名访问MetaServer获取ConfigService的地址列表
   相当于一个Eureka Proxy
   逻辑角色，和ConfigService住在一起部署（即apollo-configer有meta-service，eureka，config-service三个模块）

3. NginxLB

   和域名系统配合，协助Portal访问MetaServer获取AdminService地址列表
   和域名系统配合，协助Client访问MetaServer获取ConfigService地址列表
   和域名系统配合，协助用户访问Portal进行配置管理



## 引用/参考

[Apollo配置中心介绍 - ctripcorp - github](https://github.com/ctripcorp/apollo/wiki/Apollo配置中心介绍)

[Apollo架构体系、Apollo运行原理、Apollo配置中心简单介绍（一） - zjh_746140129 - CSDN](https://blog.csdn.net/zjh_746140129/article/details/86179522)