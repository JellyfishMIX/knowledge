# macos(类 linux)环境变量${PATH}总结



## 前言

macos 底层基于 unix 系统，因此属于类 linux 系统。关于 unix, linux, macos 的故事可以详见: [Unix, Linux 和MacOS](https://juejin.cn/post/6844903841901576199)，本文就不展开介绍啦。本文基于 macos 调试与写作，对 linux 系统有参考意义但不保证完全一致。

笔者第一次接触环境变量 ${PATH} 应该是在安装 jdk, maven, mysql 时候，在网上搜索"jdk安装", "maven 安装", "mysql 安装"等关键字，然后参考一篇篇文章调试。不明白机制，按照参考文章一顿操作，能运行成功就不想啦。做 java 开发的朋友们，大部分第一次接触环境变量应该都是这样的状态。笔者很长一段时间是这样对环境变量迷迷糊糊的状态，在后来多次安装调整环境变量时，有了一点心得总结，在本文中记录并分享。



## 环境变量是什么？

```bash
# 看一下本机配置的环境变量目录之一
cat /etc/paths
# 选其中一个看一下
cd /bin
ls
# 以下是 ls 结果。有很多熟悉的命令例如 mkdir, kill, ls, echo, rm 等命令，正因为我们把这些命令所在的目录配置进了环境变量，所以我们才能在终端中直接使用这些命令，而无需给出命令所在的目录
[         cat       cp        dash      dd        echo      expr      kill      launchctl ln        mkdir     pax       pwd       rmdir     sleep     sync      test      wait4path
bash      chmod     csh       date      df        ed        hostname  ksh       link      ls        mv        ps        rm        sh        stty      tcsh      unlink    zsh
```

查看本机的环境变量 ${PATH}

```bash
echo ${PATH}
# 笔者的机器所得到的结果。每个环境变量的值用 : 分隔开来
/usr/local/mysql/bin:/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home/bin:/usr/local/maven/apache-maven-3.6.0/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Library/Apple/usr/bin
```



## macos 环境变量加载的规则

MAC OS X环境的所有配置以及加载顺序如下：

```bash
# 系统级别
/etc/profile
/etc/paths

# 用户级别
~/.bash_profile
~/.bash_login
~/.profile

~/.bashrc（或者~/.zshrc）
```

前两个环境配置在系统启动时候就会加载，针对所有用户生效，后面四个属于具体用户级别的配置
~/.bash_profile，~/.bash_login，~/.profile依次加载，如果~/.bash_profile不存在，依次加载后面几个文件；如果~/.bash_profile文件存在，后面几个文件不会加载
~/.bashrc （或者~/.zshrc ）是bash shell打开时候加载
~/.bashrc （或者~/.zshrc）的区别                                                                        zsh终端命令工具的全局变量设置，和bashrc区别是 默认很多linux系统是base，就配置在bashrc里，如里是使用zsh 就配置在 zshrc里，zsh是比bash更强大shell



## 编写一下环境变量

以配置 jdk 环境变量举例

```bash
# 查看本机所有的 jdk
/usr/libexec/java_home -V
# 本机自行安装的 jdk 安装路径
cd /Library/Java/JavaVirtualMachines
ls
# 查看本机默认的 jdk( 默认情况下 macos 会自动选择 /Library/Java/JavaVirtualMachines 目录下版本号最高的 jdk 做为默认 jdk)
/usr/libexec/java_home
# 查看目前使用的 java 路径
which java
```

修改环境变量，执行

```bash
vim ~/.bash_profile
```

给大家展示一下我的 `~/.bash_profile` 文件

```bash
# mysql path
MYSQL_PATH=/usr/local/mysql
MYSQL_BIN=${MYSQL_PATH}/bin

# jdk path
JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
JAVA_BIN=${JAVA_HOME}/bin
CLASS_PATH=${JAVA_HOME}/lib

# maven path
M3_HOME=/usr/local/maven/apache-maven-3.6.0
M3_BIN=${M3_HOME}/bin

# homebrew 镜像源。export 导出的环境变量仅限本次终端会话有效。
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles

# 最后的 ${PATH} 是继承自父 profile 的变量。
# 小细节：${PATH} 应放在最后。因为 bash 中输入命令时，遵循就近原则，先在左边路径中找到的先用。如果 ${PATH} 写在左边，且父 profile 定义的路径中有相同名称的命令，则会执行父 profile 中定义的路径中的命令，而不会执行子 profile 中定义的路径中的命令。
PATH=${MYSQL_BIN}:${JAVA_BIN}:${M3_BIN}:${PATH}
```



## 引用/参考

[Unix, Linux 和MacOS - keith - 掘金](https://juejin.cn/post/6844903841901576199)

[macOS和Linux下source和export命令 - 布丁达人 - CSDN](https://blog.csdn.net/bym12138/article/details/104857887)

[macos设置环境变量path详解 - Mint6 - CSDN](https://blog.csdn.net/Mint6/article/details/124156340)