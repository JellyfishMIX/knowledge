# tar包、jar包、war包、tar.gz包的区别



## 文件类型不同

tar包：属于打包文件。Lniux系统上的压缩打包工具，可以将多个文件合并为一个文件，打包后的文件后缀为“tar”。简单说tar就是打包；

jar包：属于打包文件。即Java Archive的Java包。Java编译好之后生成class文件，但是如果直接发布这些class文件的不方便，所以就把许多class文件打包为一个jar包。jar包中除了class文件还包括一些资源和配置文件，通常一个jar包就是一个java程序；

war包：属于打包文件。即Web Application Archive，与jar基本相同。但通常表示一个Java的web应用程序的包。

tar.gz包：是压缩文件。经过gzip压缩后的tar文件，形成tar.gz包，扩展名为“xx.tar.gz”；

> gzip：若干种文件压缩程序的简称



## 用途不同

tar.gz包一般情况下都是源代码的安装包，需要先解压再经过编译、安装才能执行。总而言它是一个压缩文件。



## 转自

[tar包、jar包、war包、tar.gz包的区别（通过assembly打tar.gz包） - 南无南有 - CSDN](https://blog.csdn.net/qq_39416311/article/details/103097328)