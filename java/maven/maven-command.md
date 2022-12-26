# maven command



## maven 生命周期

```shell
mvn clean package -Dmaven.test.skip=true
```

参数：

- **mvn clean package依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)等７个阶段。**
- **package命令完成了项目编译、单元测试、打包功能，但没有把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库。**

```shell
mvn clean install -Dmaven.test.skip=true
```

- **mvn clean install依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install等8个阶段。**
- **install命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库，但没有布署到远程maven私服仓库。**

```shell
mvn clean deploy -Dmaven.test.skip=true
```

- **mvn clean deploy依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install、deploy等９个阶段。**
- **deploy命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库。**



## 常用命令

### 查看生效版本

在 spring boot 项目中，pom.xml 中一些 dependency 没有标注 version，如果想查看生效的 version，可以使用：

```bash
mvn help:effective-pom
```

查看每个 maven module 具体生效的 pom.mxl，里面有各个 dependency 详细的 version。

### 依赖树分析，筛选条件

```bash
# 优先使用这种方式
mvn dependency:tree
# 这种方式可能没法筛选，时间原因暂时没了解
mvn dependency:tree -Dverbose -Dincludes=commons-lang3
```

