# Maven



## maven 配置

```bash
mvn -v
```

查看maven版本信息，可以看到 maven 根目录



## 多模块项目

单模块 -> 多模块，重构

- 调整主（父）工程类型（`<packaging>`）
- 创建子模块工程（`<module>`）
  - 模型层：model
  - 持久层：persistence
  - 表示层：web



## dependencies与dependencyManagement的区别

- 父项目中的dependencies中定义的所有依赖，在子项目中都会直接继承
- 在父项目中的dependencyManagement中定义的所有依赖，子项目并不会继承，我们还要在子项目中引入我们需要的依赖，才能进行使用。此时我们在子项目中不用设置版本。



## SNAPSHOT

A large software application generally consists of multiple modules and it is common scenario where multiple teams are working on different modules of same application. For example, consider a team is working on the front end of the application as app-ui project (app-ui.jar:1.0) and they are using data-service project (data-service.jar:1.0).

Now it may happen that team working on data-service is undergoing bug fixing or enhancements at rapid pace and they are releasing the library to remote repository almost every other day.

Now if data-service team uploads a new version every other day, then following problems will arise −

- data-service team should tell app-ui team every time when they have released an updated code.
- app-ui team required to update their pom.xml regularly to get the updated version.

To handle such kind of situation, **SNAPSHOT** concept comes into play.

### What is SNAPSHOT?

SNAPSHOT is a special version that indicates a current development copy. Unlike regular versions, Maven checks for a new SNAPSHOT version in a remote repository for every build.

Now data-service team will release SNAPSHOT of its updated code every time to repository, say data-service: 1.0-SNAPSHOT, replacing an older SNAPSHOT jar.

### Snapshot vs Version

In case of Version, if Maven once downloaded the mentioned version, say data-service:1.0, it will never try to download a newer 1.0 available in repository. To download the updated code, data-service version is be upgraded to 1.1.

In case of SNAPSHOT, Maven will automatically fetch t



## Citation/Reference

[Maven - Snapshots - tutorialspoint](https://www.tutorialspoint.com/maven/maven_snapshots.htm)