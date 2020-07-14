# 使用Setfile命令修改MacOS文件创建时间(creation date)，（非touch命令），附Linux文件时间属性介绍



## 情景

有一个文件想要修改“创建时间”和“修改时间”：

<img src="https://image-hosting.jellyfishmix.com/20200623162822.png" alt="待修改的文件" style="zoom:50%;" />



网上普遍使用类Unix系统的命令`touch`来实现（预先说明，此普遍方法无法满足修改“创建时间”的需求。后文有使用`Setfile`命令这一可行的解决方法）：

### 命令格式

```bash
touch [选项参数] <文件名>
```

### 命令参数

- `-t` 使用指定的日期时间，修改文件的“atime（访问时间）”，“mtime（修改时间）“。

- `-a` 或`--time=atime`或`--time=access`或`--time=use` 只修改atime（访问时间）。
- `-m` 或`--time=mtime`或`--time=modify` 只修改mtime（修改时间）。

- `-c` 或`--no-create` 不建立任何文档，此参数将修改“atime（访问时间）”，“mtime（修改时间）“，“ctime（文件属性变更时间）”。

- `-d` 使用指定的日期时间，而非现在的时间。

- `-f` 此参数将忽略不予处理，仅负责解决BSD版本touch指令的兼容性问题。

- `-r` 把指定文档或目录的日期时间，统统设成和参考文档或目录的日期时间相同。

关于Linux系统的atime（访问时间），mtime（修改时间），ctime（文件属性变更时间），后文的“附录”部分有介绍。

### e.g.

```bash
// 使用指定的时间（2020年02月03日12:30），修改文件的“atime（访问时间）”，“mtime（修改时间）“
touch -t 202002031230 <文件名>
// 使用指定的时间（2020年02月03日12:30），修改文件的“修改时间”
touch -mt 202002031230 <文件名>
```

不论是`touch -t`还是`touch -mt`，执行完毕后：

<img src="https://image-hosting.jellyfishmix.com/20200623171536.png" alt="使用touch -t命令，仅修改了修改时间" style="zoom:50%;" />



仅修改了“修改时间”，但“创建时间”还是没有被修改。

原因是：**`touch -t` 仅会当 指定的时间 在 原始创建时间 之前时，才会修改创建时间**。



## 解决方法

使用`Setfile`命令。

`Setfile`命令是一个MacOS X的开发者工具，它可以修改文件的creation（创建时间）和modification date（修改时间）。不过前提是，你的MacOS上必须已经安装了Xcode。如果没有的话，可以去Mac App Store安装。你可以在`/usr/bin/SetFile`位置找到此命令行工具。

### 使用方法

```bash
Setfile -d '01/10/2020 11:00:00' <文件名>
```

执行后：

<img src="https://image-hosting.jellyfishmix.com/20200623174227.png" alt="使用Setfile命令，文件的创建时间修改成功" style="zoom:50%;" />



文件的“创建时间”修改成功！



## 附录

### Linux | 文件的时间属性

在Linux系统下，文件的时间属性主要分为三种：

#### atime（访问时间）：

也就是Access time。读一次文件的内容，该文件的atime就会更新。比如常见的使用more、cat对该文件进行查看时，其atime将更新。

#### mtime（修改时间）：

也就是Modify time。对该文件进行内容上的修改，该文件的mtime就会更新。比如常见的使用vi、vim对文件进行修改后保存，其mtime将更新。

#### ctime（文件属性变更时间）：

也就是Change time。对该文件的属性状态进行修改，改文件的ctime就会更新。比如文件名、内容、大小、权限、所属组等改变时，其ctime将更新。



#### ll或ls命令查看文件的时间属性

- `ll --time=atime`或`ls -lu`命令查看atime（访问时间）

  <img src="https://image-hosting.jellyfishmix.com/20200623175157.png" alt="`ll --time=atime`或`ls -lu`命令查看atime（访问时间）" style="zoom:67%;" />

- `ll`或`ls -l`命令查看mtime（修改时间）

  <img src="https://image-hosting.jellyfishmix.com/20200623174900.png" alt="`ll`或`ls -l`命令查看mtime（修改时间）" style="zoom:67%;" />

- `ll --time=ctime`或`ls -lc`命令查看ctime（文件属性变更时间）

  <img src="https://image-hosting.jellyfishmix.com/20200623175818.png" alt="`ll --time=ctime`或`ls -lc`命令查看ctime（文件属性变更时间）" style="zoom:67%;" />



#### stat命令查看文件的时间属性

可以使用`stat`命令同时查看文件的三种属性

![stat命令查看文件的时间属性](https://image-hosting.jellyfishmix.com/20200623180631.png)



#### find命令查找特定时间要求的文件

结合find命令可以查找特定时间要求的文件，例如查询最近24小时内修改过的文件：

```bash
find ./ -mtime -24
```

<img src="https://image-hosting.jellyfishmix.com/20200623181435.png" alt="find命令查找特定时间要求的文件" style="zoom:67%;" />



#### 文件创建时间

Linux常见的文件系统，没有文件创建时间属性，关于这一点，可以去网上看相关讨论。



## 引用

[Linux | 文件的时间属性 - 嘉为科技的文章 - 知乎](https://zhuanlan.zhihu.com/p/108055568)