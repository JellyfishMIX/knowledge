# Maven 项目命名规范（转）



## Guide to naming conventions on groupId, artifactId and version

- groupId

   will identify your project uniquely across all projects, so we need to enforce a naming schema. It has to follow the package name rules, what means that has to be at least as a domain name you control, and you can create as many subgroups as you want. Look at 

  More information about package names

  .

   

  eg. `org.apache.maven`, `org.apache.commons`

  A good way to determine the granularity of the `groupId` is to use the project structure. That is, if the current project is a multiple module project, it should append a new identifier to the parent's `groupId`.

  eg. `org.apache.maven`, `org.apache.maven.plugins`, `org.apache.maven.reporting`

- artifactId

   is the name of the jar without version. If you created it then you can choose whatever name you want with lowercase letters and no strange symbols. If it's a third party jar you have to take the name of the jar as it's distributed.

   

  eg. `maven`, `commons-math`

- version

   if you distribute it then you can choose any typical version with numbers and dots (1.0, 1.1, 1.0.1, ...). Don't use dates as they are usually associated with SNAPSHOT (nightly) builds. If it's a third party artifact, you have to use their version number whatever it is, and as strange as it can look.

   

  eg. `2.0`, `2.0.1`, `1.3.1`

以上内容是maven官网文档中的，命名约定指南（https://maven.apache.org/guides/mini/guide-naming-conventions.html）

总的来说：



groupId:定义当前Maven项目隶属的实际项目，例如 org.sonatype.nexus，此id前半部分org.sonatype代表此项目隶属的组织或公司，后部分代表项目的名称，如果此项目多模块话开发的话就子模块可以分为org.sonatype.nexus.plugins和org.sonatype.nexus.utils等。 特别注意的是groupId不应该对应项目隶属的组织或公司，也就是说groupId不能只有org.sonatype而没有nexus。

 例如：我建立一个项目，此项目是此后所有项目的一个总的平台，那么groupId应该是org.limingming.projectName,projectName是平台的名称，org.limingming是代表我个人的组织，如果以我所在的浪潮集团来说的话就应该是com.inspur.loushang。

artifactId是构件ID，该元素定义实际项目中的一个Maven项目或者是子模块，如上面官方约定中所说，构建名称必须小写字母，没有其他的特殊字符，推荐使用“实际项目名称－模块名称”的方式定义，例如：spirng-mvn、spring-core等。

1. 推荐格式：使用实际项目名称作为artifactId的前缀，紧接着为模块名称。
2. 举例：nexus-indexer、spring-mvc、hibernate-c3po……这些id都是以实际项目名称作为前缀，然后接着一个中划线，再紧跟项目的模块名称，默认情况下maven会在artifactId添加version作为最后生成的名称。例如：spirng-mvn-2.0.0.jar。



## Citation/Reference

[Maven项目命名规范 - 编程之间 - CSDN](https://blog.csdn.net/limm33/article/details/60959044)