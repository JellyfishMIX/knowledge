# 文件



## 基本知识

### 新建文件夹

```bash
mkdir xxx
```

- `-p` 确保目录名称存在，不存在的就建一个。

e.g.

```bash
mkdir -p /opt/settings/
```

### 显示隐藏文件

```bash
ls -la
```

### 复制、移动与删除

####  文件复制命令cp

```bash
cp [-adfilprsu] 源文件(source) 目标位置(destination)
cp [option] source1 source2 source3 ... directory
```

##### option

- `-a` 是指archive的意思，也说是指复制所有的目录。
- `-d` 若源文件为连接文件(link file)，则复制连接文件属性而非文件本身。
- `-f` 强制(force)，若有重复或其它疑问时，不会询问用户，而强制复制。
- `-i` 若目标文件(destination)已存在，在覆盖时会先询问是否真的操作。
- `-l` 建立硬连接(hard link)的连接文件，而非复制文件本身。
- `-p` 与文件的属性一起复制，而非使用默认属性。
- `-r` 递归复制，用于目录的复制操作。
- `-s` 复制成符号连接文件(symbolic link)，即“快捷方式”文件。
- `-u` 若目标文件比源文件旧，更新目标文件。

#### 文件移动命令mv

```bash
mv [-fiv] source destination
```

##### option

- `-f` force，强制直接移动而不询问。
- `-i` 若目标文件(destination)已经存在，就会询问是否覆盖。
- `-u` 若目标文件已经存在，且源文件比较新，才会更新。

####文件删除命令rm

```bash
rm [fir] 文件或目录
```

##### option

- `-f` 强制删除。
- `-i` 交互模式，在删除前询问用户是否操作。
- `-r` 递归删除，常用在目录的删除。

### 打包

```bash
tar -cvf <filename>.tar <filename>
```

此命令只是打包，不是压缩。

#### option

- `-c` 或 `--create` 建立新的备份文件。

- `-r` 或 `--append` 新增文件到已存在的备份文件的结尾部分。

- `-v` 显示指令执行过程。
- `-f <备份文件>` 或 `--file=<备份文件>` 指定备份文件。
- `-z` 或 `--gzip` 或 `--ungzip` 通过gzip指令处理备份文件。

### 解压

```bash
tar -zxvf ./xxx -C /xxx
```

tar是用来建立，还原备份文件的工具程序，它可以加入，解开备份文件内的文件。

- `-z` 通过gzip指令处理备份文件。
- `-x` 从备份文件中还原文件。

- `-v` 显示指令执行过程。

- `-f` 指定备份文件。

- `-C` <目的目录> 切换到指定目录。

### tail 动态实时展示文件的变化过程

tail命令可以动态实时展示文件的变化过程。

**example**: 

```bash
tail -f error.2019-12-28.log
```

```bash
tail -f info.2019-12-28.log
```

### 查看磁盘剩余空间

```bash
df -hl
```

### 查看当前文件夹大小

```bash
du -sh
```

###  列出当前文件夹下的所有文件夹及其大小，并按照文件夹大小排序

```bash
du -sh * | sort -n
```

### 文件描述符

![img](https://image-hosting.jellyfishmix.com/20200718110444.jpg)

#### 文件描述符基本知识

> 文件描述符是与文件输入、输出关联的整数。它们用来跟踪已打开的文件。最常见的文件描述符是stidin、stdout、和stderr。我们可以将某个文件描述符的内容重定向到另外一个文件描述符中。 -- 《linux shell脚本攻略》


文件描述符我们常见的就是系统预留的0，1和2这三个，他们的意义分别有如下对应关系：

- 0 -- stdin（标准输入）
- 1 -- stdout （标准输出）
- 2 -- stderr （标准错误）

其中，shell编程里经常用到的就是描述符1，和描述符2。这样下面我们来举两个栗子，就知道神马是1和2了：

1. stdout
   假设：在当前目录下我们“有且只有”一个文件名为 123.txt 的文本文件。这个时候我们运行下面的命令【ls 123.txt】:

   ![img](https://image-hosting.jellyfishmix.com/20200718110630.jpg)

2. stderr

   按照上面同样的假设，我们运行另外一跳命令【ls abc.txt】：

   ![img](https://image-hosting.jellyfishmix.com/20200718110701.jpg)

   我们就会获得一个标准错误stderr的输出结果“ls：无法访问abc.txt：没有那个文件或目录”。

   有同学应该会觉得，这两个事例好像跟1和2这两个阿拉伯数字好像没有关系。这个就要结合第二个知识点“重定向操作”来理解了。

#### 重定向操作

重定向操作，其实就是通过在shell命令后面追加一个重定向操作符号，将shell命令对应的文件描述符输出的文本信息重新输入到另外一个指定文件的操作。

重定向操作符号有两个>和>>。尽管这两个操作符都可以将重定向到文件，但是前者会先清空文件，再写入内容；后者会将内容追加到现有文件的尾部。（对了，重定向的操作制定的文件如果原来不存在的话，重定向的操作会主动创建这个文件名的文件的）

- 重定向标准输出stdout

  ![img](https://image-hosting.jellyfishmix.com/20200718110935.jpg)

  如上图所示，对比没有添加重定向的操作，ls命令在使用之后并没有将字符“123.txt”这个字符串打印到屏幕上。在紧接着的cat操作之后，我们可以看到本来应该输出字符串被记录在了stdout.txt这个文件里面了。

  

  其实，对于标准输出的重定向操作，>等同于1>。上面栗子执行命令 `ls 123.txt > stdout.txt` 得到的效果也是一样的。

- 重定向标准错误stderr

  ![img](https://image-hosting.jellyfishmix.com/20200718111023.jpg)

  如上图所示，文件描述符2，标准错误的重定向也是同样的原理被记录在了文件stderr.txt这个文件里面了。

  

  描述符的重定向还有下面的几种用法：
  你可以将stderr单独定向到一个文件，将stdout重定向到另一个文件：

  ```bash
  cmd 2>stderr.txt 1>stdout.txt
  ```

  也可以利用下面的方法，将stderr转换成stdout，使得stderr和stdout都被重新定向到同一个文件中：

  ```bash
  cmd> output.txt 2>&1
  ```

  或者采用这个方法

  ```bash
  # 两个表达式效果一样
  cmd &> output.txt
  cmd >& output.txt
  ```

####linux特殊文件

> /dev/null是一个特殊的设备文件，这个文件接收到的任何数据都会被丢弃。因此，null这个设备通常也被成为位桶（bit bucket）或黑洞。 --  《linux shell脚本攻略》

简单地理解就是，重定向操作给这个/dev/null文件的所有东西都会被丢弃。



## 引用

[linux如何复制文件夹和移动文件夹](https://www.cnblogs.com/liaojie970/p/6746230.html)

[shell程序中 2> /dev/null 代表什么意思？ - 裕用ID的回答 - 知乎](https://www.zhihu.com/question/53295083/answer/135258024)