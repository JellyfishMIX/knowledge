## Maven 打包

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

