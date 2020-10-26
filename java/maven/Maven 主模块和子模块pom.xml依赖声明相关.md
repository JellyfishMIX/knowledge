# Maven 主模块和子模块pom.xml依赖声明相关（依赖放入子模块还是父模块）



## 前言

今天想到了一个问题，如果一个依赖只有子模块用到了，是放入子模块的 pom.xml 呢，还是放入父模块的 pom.xml 呢？

理论上当然是子模块单独声明更符合逻辑。但是以上问题的场景来源有两个：

1. 为了方便，或者考虑到其它子模块或许以后会用到此依赖的可能性。
2. 单模块项目改造为多模块后，原本的依赖全部声明在父模块 pom.xml 中，考虑是否要大量迁移到用到的子模块中。

进而引申出的问题：

如果依赖全部放入父模块，部分子模块没有用到这些依赖，是否会增加这些子模块打包后的代码体积？



## 背景知识

### dependencies与dependencyManagement的区别

- 父项目中的 `<dependencies></dependencies>` 中定义的所有依赖，在子项目中都会直接继承。
- 在父项目中的 `<dependencyManagement></dependencyManagement>` 中定义的所有依赖，子项目并不会继承，我们还要在子项目中引入我们需要的依赖，才能进行使用。此时我们在子项目中不用设置版本。



## 实验

为了回答这个问题：“如果依赖全部放入父模块，部分子模块没有用到这些依赖，是否会增加这些子模块打包后的代码体积？”。我们拿一个 maven 多模块项目打包测试一下。

实验材料：

![image-20201025225704040](https://image-hosting.jellyfishmix.com/20201025225704.png)



如图，一个多模块项目。

其中 wx-common 模块只是放了一些 enums：

![image-20201025225821556](https://image-hosting.jellyfishmix.com/20201025225821.png)



父模块依赖：

```xml
<properties>
    <java.version>11</java.version>
    <spring-cloud.version>Hoxton.SR8</spring-cloud.version>
    <wx-common-version>0.0.1-SNAPSHOT</wx-common-version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.2.6.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <version>2.2.5.RELEASE</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>2.2.6.RELEASE</version>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!--lombok-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.12</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.json/json -->
    <dependency>
        <groupId>org.json</groupId>
        <artifactId>json</artifactId>
        <version>20190722</version>
    </dependency>

    <dependency>
        <groupId>com.jellyfishmix.interchange</groupId>
        <artifactId>common</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <version>2.2.2.RELEASE</version>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.jellyfishmix.interchange</groupId>
            <artifactId>wx-common</artifactId>
            <version>${wx-common-version}</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```



wx-common 模块无单独引入的依赖。

wx-common 模块单独打包后的大小（3982 bytes）：

![Screen Shot 2020-10-26 at 4.05.44 PM](https://image-hosting.jellyfishmix.com/20201026160842.png)



接下来我们把父模块的依赖都放入 `<dependencyManagement></dependencyManagement>` 中，这样子模块就不会全部继承这些依赖，而是需要在子模块的 pom.xml 中也进行声明，子模块才能继承对应的依赖。

按照博主的猜想，**子模块最初继承了很多父模块的依赖，当单独打包子模块时，这些依赖被打入了子模块jar包中。而这些继承过来的父模块的依赖中，有很多是子模块不需要的，因此子模块单独打出的包，会有不少冗余体积**。

我们把父模块的依赖都挪入 `<dependencyManagement></dependencyManagement>` 中，而**子模块又没有在自己的 pom.xml 中声明这些依赖，也就不会继承这些依赖，这样子模块单独打出的包，会不会减少很多体积呢**？

按我们的推测，把父模块的依赖都放入 `<dependencyManagement></dependencyManagement>` 中，然后对子模块单独打包（3982 bytes）：

![Screen Shot 2020-10-26 at 4.07.27 PM](https://image-hosting.jellyfishmix.com/20201026161539.png)



可以看到打包出来的 jar，并没有按照我们预先设想的，体积减少了很多，而是和之前的体积一模一样（都是3982 bytes）。

看到这个结果，博主百思不得其解。难道**子模块继承的父模块的依赖，如果在子模块中没有被使用，在子模块单独打包时，就不会被打入 jar **吗？

我们再做一个实验来验证猜想，现在父模块的依赖还是在 `<dependencyManagement></dependencyManagement>` 中，需要在子模块的 pom.xml 中也进行声明，子模块才能继承对应的依赖。我们给子模块的 pom.xml 多声明几个依赖：

```xml
<!--lombok-->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.12</version>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>2.2.5.RELEASE</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.json/json -->
<dependency>
    <groupId>org.json</groupId>
    <artifactId>json</artifactId>
    <version>20190722</version>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

然后对子模块单独打包（4175 bytes）：

![Screen Shot 2020-10-26 at 4.22.50 PM](https://image-hosting.jellyfishmix.com/20201026162340.png)



可以看到我们的 jar 包体积确实增加了（4175 - 3982 = 193 bytes），但这些增加的代码体积，应该是我们的 pom.xml 中新增的一对对 `<dependency></dependency>` 的体积，而不是真正引入的依赖的代码。

因此，博主确信了推测：**子模块继承的父模块的依赖/子模块声明的依赖，如果在子模块中没有被使用，在子模块单独打包时，就不会被打入 jar **。

进一步实验来确认推测，我们在子模块中使用一下声明的依赖。只在子模块中加两个注解：`@FeignClient(name = "interchange-wx")`，对子模块单独打包（4259 bytes）：

![Screen Shot 2020-10-26 at 5.13.07 PM](https://image-hosting.jellyfishmix.com/20201026232047.png)

打包结果（4259 - 4175 = 84 bytes）。

因此，maven 打包加入的依赖代码应该是被调用到的部分代码，没有被调用到的依赖代码不会被加入打包后的 jar 包中。



## 实验结论

1. 子模块继承的父模块的依赖/子模块声明的依赖，如果在子模块中没有被使用，在子模块单独打包时，就不会被打入 jar 。

2. maven 打包加入的依赖代码是被调用到的部分代码，没有被调用到的依赖代码不会被加入打包后的 jar 包中。



## 推荐做法

对于 “依赖放入子模块还是父模块” 这个问题，推荐将依赖放入父模块的  `<dependencyManagement></dependencyManagement>` 中，然后子模块有需要的依赖，在子模块的 pom.xml 中声明。这样便于在父模块中统一管理依赖版本，避免子模块依赖版本不一致造成的混乱或冲突。