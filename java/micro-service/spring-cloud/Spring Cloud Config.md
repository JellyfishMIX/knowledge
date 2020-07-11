# Spring Cloud Config



## 使用原因

- 方便维护
- 配置内容安全与权限
- 更新配置项目避免重启



## Architecture（架构）

![截屏2020-06-22下午11.05.46](https://image-hosting.jellyfishmix.com/20200622230610.png)

- 从远端git仓库拉取配置代码。
- 会把代码存入本地git仓库，当远端仓库不能拉取时，会从本地仓库拉取。
- 业务服务使用`config-client`与`config-server`进行交互。



## 配置文件命名规则

`/{name}-{profiles}.yml`

`/{label}/{name}-{profiles}.yml`



- `label` 分支

- `name` 服务名

- `profiles` 环境



## 配合Spring Cloud Bus自动刷新配置

### 示意图

![配合Spring Cloud Bus自动刷新配置](https://image-hosting.jellyfishmix.com/20200626024442.png)

### pom.xml依赖

config-server

```xml
<!--通知config-client拉取新配置-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

config-client

```xml
<!--作为config-client拉取新配置-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

### bootstrap.yml配置

需要暴露/actuator/bus-refresh这个endpoint，设置为暴露全部endpoints

```yaml
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
        # 暴露全部endpoints
        # include: "*"
```

手动刷新配置请求路径

```
method: POST
path: /actuator/bus-refresh
```

```bash
curl -v -X POST "http://localhost:8080/actuator/bus-refresh"
```

- `-v` 输出通信的整个过程，用于调试。
- `-X` 指定 HTTP 请求的方法。

### Webhook实现自动刷新

git仓库工具webhook自动刷新请求路径

```
/monitor
```

`content type`设置为`application/json`

![截屏2020-06-26上午2.33.19](https://image-hosting.jellyfishmix.com/20200626024608.png)

### @RefreshScope

在配置变更后，需要同步使用最新配置的地方，加上@RefreshScope注解。如果不加，即使config-client拉取了最新配置，也没法应用最新的配置。

application.yml

```yml
# Spring Cloud Bus演示使用
girl:
  name: lili
  age: 19
```

GirlConfig.java

```java
@Data
@Component
@ConfigurationProperties("girl")
@RefreshScope
public class GirlConfig {
    private String name;
    private Integer age;
}
```

GirlController.java

```java
@RestController
public class GirlController {
    @Autowired
    private GirlConfig girlConfig;

    @GetMapping("/girl/print")
    public String print() {
        return "name: " + girlConfig.getName() + "\nage: " + girlConfig.getAge();
    }
}
```



## 注意事项

SpringCloud有一个“启动上下文”，主要是用于加载远程的配置，也就是加载ConfigServer里面的配置，默认加载顺序为:

1. 加载bootstrap.里面的配置 
2. 链接config server，加载远程配置
3. 加载application.*里面的配置

因此，需要借助于"启动上下文"来处理加载远程配置，将项目本身的`application.yml`/`application.properties`改名为`bootstrap.yml`/`bootstrap.properties`。