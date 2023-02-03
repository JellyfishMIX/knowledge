# linux-command



## 电源

### 立即关机

```bash
sudo halt
```



## 进程，内存

### 查看内存占用情况

`top`

退出查看

`q`

``htop``

退出查看

`q`

### 查找进程名字

```bash
ps -ef | grep
```

ps 命令的作用是显示进程信息，| 符号，是个管道符号，表示ps 和 grep 命令同时执行

grep 命令是查找(Global Regular Expression 
Print)，能使用正则表达式搜索文本，然后把匹配的行显示出来

**arguments**:

-e	显示所有进程
-f	全格式
-h	不显示标题
-l	长格式
-w	宽输出
-a	显示终端上的所有进程，包括其他用户的进程
-r	只显示正在运行的进程
-u	以用户为主的格式来显示程序状况
-x	显示所有程序，不以终端机来区分

**response**:

UID	程序被该 UID 所拥有

PID	就是这个程序的 ID 

PPID	则是其上级父程序的ID

C	CPU使用的资源百分比

STIME	系统启动时间

TTY	登入者的终端机位置

TIME	使用掉的CPU时间。

CMD	所下达的是什么指令

**example**

```bash
ps -ef | grep nginx
```

| UID       | PID   | PPID  | C    | STIME | TTY   | TIME     | CMD                                          |
| --------- | ----- | ----- | ---- | ----- | ----- | -------- | -------------------------------------------- |
| root      | 30890 | 1     | 0    | 13:36 | ?     | 00:00:00 | **nginx**: master process /usr/sbin**nginx** |
| **nginx** | 30891 | 30890 | 0    | 13:36 | ?     | 00:00:00 | **nginx**: worker process                    |
| root      | 31222 | 31202 | 0    | 17:46 | pts/0 | 00:00:00 | grep --color=auto **nginx**                  |

### 根据端口号查找进程

```bash
lsof [参数][文件]
```

lsof（list open files）是一个列出当前系统打开文件的工具。任何事物都以文件的形式存在，通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。

**arguments**:

-i	<条件> 列出符合条件的进程。（4、6、协议、:端口、 @ip ）

e.g.

```bash
lsof -i tcp:8080
或
lsof -i:8080
```

找到后，杀死进程：

```bash
kill -9 <PID>
```

### 列出所有开放的端口

```bash
netstat -tulpn
```

### 关闭某个进程

```bash
kill -9 <PID>
```

- kill命令用于删除执行中的程序或工作。
- 数字为信号。
- kill可将指定的信号送至程序。预设的信号为SIGTERM(15)，可将指定程序终止。若仍无法终止该程序，可使用SIGKILL(9)信息尝试强制删除程序。
- 程序或工作的PID可利用ps指令或jobs指令查看。

- 默认的 `kill` 是 `kill -15`。
  - `kill -15` 代表的信号为 `SIGTERM`，这是告诉进程"需要被关闭，请自行停止运行并退出"。
  - 而 `kill -9` 代表的信号是 `SIGKILL`，表示进程被终止，需要立即退出。



### systemd

systemctl

/lib/systemd/

### 在后台持续运行进程

```bash
nohup command > path 2>&1 &
```

- `nohup` 表示不挂起地。

- `command` 启动要执行的进程。

- `> path` 表示把path作为标准输出路径。

- `2>&1` `2`表示错误输出，`2>` 表示重定向操作错误提示信息，`&1`表示标准输出路径。意思是把错误输出也输出到刚刚定义的标准输出路径。

  linux中有三种标准输入输出，分别是STDIN，STDOUT，STDERR，对应的数字是0，1，2。

  STDIN是标准输入，默认从键盘读取信息；

  STDOUT是标准输出，默认将输出结果输出至终端；

  STDERR是标准错误，默认将输出结果输出至终端。

  由于STDOUT与STDERR都会默认显示在终端上，为了区分，就有了编号的0，1，2的定义，用1表示STDOUT，2表示STDERR。

  2>&1，指将标准输出、标准错误指定为同一输出路径。

  关于2>&1: 

  ```
  File descriptor 1 is the standard output (stdout).
  File descriptor 2 is the standard error (stderr).
  
  Here is one way to remember this construct (although it is not entirely accurate): at first, 2>1 may look like a good way to redirect stderr to stdout. However, it will actually be interpreted as "redirect stderr to a file named 1". & indicates that what follows and precedes is a file descriptor and not a filename. So the construct becomes: 2>&1.
  
  Consider >& as redirect merger operator.
  ```

- `&` 表示在后台运行。

e.g.

```bash
nohup java -jar ./sell-eureka-server.jar > /dev/null 2>&1 &
```

- `/dev/null` 

  linux特殊文件

  > /dev/null是一个特殊的设备文件，这个文件接收到的任何数据都会被丢弃。因此，null这个设备通常也被成为位桶（bit bucket）或黑洞。 --  《linux shell脚本攻略》

  简单地理解就是，重定向操作给这个/dev/null文件的所有东西都会被丢弃。

### 关闭后台运行的进程

```
ps -ef | grep process-name
(then you can get pid of the specified process in the output)
kill -9 pid
```

### ps -eo 查看某进程的开始时间、执行时长

ps -eo lstart 启动时间

ps -eo etime 运行多长时间.

ps -eo pid,lstart,etime | [grep](https://so.csdn.net/so/search?q=grep&spm=1001.2101.3001.7020) 66510

这行命令含义：查看pid=66510的进程，其启动时间和运行时间是什么

查看进程信息更多命令请见：[linux查看某进程的开始时间、执行时长](https://blog.csdn.net/weixin_41712499/article/details/120055391)



## 网络

### 查询外网IP

```bash
curl ifconfig.me
```

### 编辑路由表

```bash
vi /etc/hosts
```

### weget 根据URL下载

```bash
wget [参数] [URL地址]
```

wget 命令用来从指定的URL下载文件。wget非常稳定，它在带宽很窄的情况下和不稳定网络中有很强的适应性，如果是由于网络的原因下载失败，wget会不断的尝试，直到整个文件下载完毕。如果是服务器打断下载过程，它会再次联到服务器上从停止的地方继续下载。这对从那些限定了链接时间的服务器上下载大文件非常有用。

**arguments**:

-O –output-document=FILE	把文档写到FILE文件中

### 追踪域名路由

```bash
traceroute
```

### scp

下载

```shell
scp root@ip:/path /localPath
```




```shell
#!/bin/sh
read name 
echo "$name It is a test"
```

以上代码保存为 test.sh，name 接收标准输入的变量，结果将是: 

```shell
[root@www ~]# sh test.sh
OK                     #标准输入
OK It is a test        #输出
```

### curl

#### 下载文件

利用 curl 下载文件。
使用内置 option：-o(小写)

```bash
curl -o dodo1.jpg http:www.linux.com/dodo1.JPG
```

使用内置 option：-O(大写)

```bash
curl -O http://www.linux.com/dodo1.JPG
```

这样就会以服务器上的名称保存文件到本地

#### 将 curl 输出保存到文件

我们可以使用 -o/-O 选项将 curl 命令的结果保存到文件中。

- -o（小写 o）结果将保存在命令行中提供的文件名中
- -O（大写O）URL中的文件名将被用作存储结果的文件名

```bash
curl -o mygettext.html http://www.gnu.org/software/gettext/manual/gettext.html
```

#### 循环下载

有时候下载图片可以能是前面的部分名称是一样的，就最后的尾椎名不一样

```bash
curl -O http://www.linux.com/dodo[1-5].JPG
```

这样就会把 dodo1，dodo2，dodo3，dodo4，dodo5 全部保存下来

#### 下载重命名

```bash
curl -O http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

由于下载的 hello 与 bb 中的文件名都是 dodo1，dodo2，dodo3，dodo4，dodo5。因此第二次下载的会把第一次下载的覆盖，这样就需要对文件进行重命名。

```bash
curl -o #1_#2.JPG http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

这样在 hello/dodo1.JPG 的文件下载下来就会变成 hello_dodo1.JPG，其他文件依此类推，从而有效的避免了文件被覆盖

#### 分块下载

有时候下载的东西会比较大，这个时候我们可以分段下载。使用内置 option：-r

```bash
curl -r 0-100 -o dodo1_part1.JPG http://www.linux.com/dodo1.JPG
curl -r 100-200 -o dodo1_part2.JPG http://www.linux.com/dodo1.JPG
curl -r 200- -o dodo1_part3.JPG http://www.linux.com/dodo1.JPG
cat dodo1_part* > dodo1.JPG
```

这样就可以查看 dodo1.JPG 的内容了

#### 通过ftp下载文件

curl可以通过ftp下载文件，curl提供两种从ftp中下载的语法

```bash
curl -O -u 用户名:密码 ftp://www.linux.com/dodo1.JPG
curl -O ftp://用户名:密码@www.linux.com/dodo1.JPG
```

#### 显示下载进度条

```bash
curl -# -O http://www.linux.com/dodo1.JPG
```

#### 不会显示下载进度信息

```bash
curl -s -O http://www.linux.com/dodo1.JPG
```

### 防火墙

centos 7 防火墙相关命令，centos 7 请使用 systemctl 命令

#### 查看防火墙状态

```bash
systemctl status firewalld
service iptables status
```

#### 暂时关闭防火墙

```bash
systemctl stop firewalld
service iptables stop
```

#### 永久关闭防火墙

```bash
systemctl disable firewalld
chkconfig iptables off
```

#### 重启防火墙

```bash
systemctl enable firewalld
service iptables restart
```

#### 永久关闭后重启

```bash
# 暂时还没有试过
chkconfig iptables on
```



## 文件

### 新建文件夹

```bash
mkdir xxx
```

- `-p` 可以创建多层目录。确保目录名称存在，不存在的就自动建一个。

e.g.

```bash
mkdir -p /opt/settings/confg
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

option

- `-a` 是指archive的意思，也说是指复制所有的目录。
- `-d` 若源文件为连接文件(link file)，则复制连接文件属性而非文件本身。
- `-f` 强制(force)，若有重复或其它疑问时，不会询问用户，而强制复制。
- `-i` 若目标文件(destination)已存在，在覆盖时会先询问是否真的操作。
- `-l` 建立硬连接(hard link)的连接文件，而非复制文件本身。
- `-p` 与文件的属性一起复制，而非使用默认属性。
- `-r` 递归复制，用于目录的复制操作。
- `-s` 复制成符号连接文件(symbolic link)，即“快捷方式”文件。
- `-u` 若目标文件比源文件旧，更新目标文件。

#### 文件移动命令/文件重命名 mv

此方式同样对目录适用，因为在 linux 中目录也是文件

```bash
mv [-fiv] source destination
# e.g.
mv apache-zookeeper-3.8.0-bin zookeeper-3.8.0
```

option

- `-f` force，强制直接移动而不询问。
- `-i` 若目标文件(destination)已经存在，就会询问是否覆盖。
- `-u` 若目标文件已经存在，且源文件比较新，才会更新。

####文件删除命令rm

```bash
rm [fir] 文件或目录
```

option

- `-f` 强制删除。
- `-i` 交互模式，在删除前询问用户是否操作。
- `-r` 递归删除，常用在目录的删除。

### 打包

```bash
tar -cvf <filename>.tar <filename>
```

`<filename>` 也可以是文件夹名称。

此命令只是打包，不是压缩。

option

- `-c` 或 `--create` 建立新的备份文件。

- `-r` 或 `--append` 新增文件到已存在的备份文件的结尾部分。

- `-v` 显示指令执行过程。
- `-f <备份文件>` 或 `--file=<备份文件>` 指定备份文件。
- `-z` 或 `--gzip` 或 `--ungzip` 通过gzip指令处理备份文件。

### tar 解压

```bash
tar -zxvf ./xxx -C /xxx
# e.g.
tar -zxvf /root/home/apache-zookeeper-3.8.0-bin.tar.gz -C /root/programming/server
```

tar是用来建立，还原备份文件的工具程序，它可以加入，解开备份文件内的文件。

- `-z` 通过gzip指令处理备份文件。
- `-x` 从备份文件中还原文件。
- `-v` 显示指令执行过程。
- `-f` 指定备份文件。
- `-C` <目的目录> 切换到指定目录。

### 文件权限

#### chmod

```bash
chmod -R 000 /
```

Linux/Unix 的文件调用权限分为三级 : 文件拥有者(User)、群组(Group)、其他(Other)。利用 chmod 可以藉以控制文件如何被他人所调用。

**parameters**: 

- -R : 对目前目录下的所有文件与子目录进行相同的权限变更
- 数字是二进制的十进制表示。r 表示可读取, w表示可写入, x 表示可执行，其中：r=4, w=2, x=1。000所在三位分别表示：文件拥有者(User)、群组(Group)、其他(Other)的权限。其中每一位：
  - 若要rwx属性则4+2+1=7
  - 若要rw-属性则4+2=6
  - 若要r-x属性则4+1=5
  - 0表示：---属性，即rwx权限都取消
  - 000表示：文件拥有者(User)、群组(Group)、其他(Other)的rwx权限都取消
- / 表示系统根目录，即从根目录开始

### tail

-n<行数> 显示文件的尾部 n 行内容

tail命令可以动态实时展示文件的变化过程。

**example**: 

```bash
tail -f error.2019-12-28.log
```

```bash
tail -f info.2019-12-28.log
```

注：需要添加 -f 才可动态实时展示。

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

####linux 特殊文件

> /dev/null是一个特殊的设备文件，这个文件接收到的任何数据都会被丢弃。因此，null这个设备通常也被成为位桶（bit bucket）或黑洞。 --  《linux shell脚本攻略》

简单地理解就是，重定向操作给这个/dev/null文件的所有东西都会被丢弃。

### ln 命令

Linux ln（英文全拼：link files）命令是一个非常重要命令，它的功能是为某一个文件在另外一个位置建立一个同步的链接。

当我们需要在不同的目录，用到相同的文件时，我们不需要在每一个需要的目录下都放一个必须相同的文件，我们只要在某个固定的目录，放上该文件，然后在 其它的目录下用ln命令链接（link）它就可以，不必重复的占用磁盘空间。

#### 语法

```
 ln [参数][源文件或目录][目标文件或目录]
```

其中参数的格式为

[-bdfinsvF] [-S backup-suffix] [-V {numbered,existing,simple}]

[--help] [--version] [--]

#### 命令功能
Linux文件系统中，有所谓的链接(link)，我们可以将其视为档案的别名，而链接又可分为两种 : 硬链接(hard link)与软链接(symbolic link)，硬链接的意思是一个档案可以有多个名称，而软链接的方式则是产生一个特殊的档案，该档案的内容是指向另一个档案的位置。硬链接是存在同一个文件系统中，而软链接却可以跨越不同的文件系统。

不论是硬链接或软链接都不会将原本的档案复制一份，只会占用非常少量的磁碟空间。

软链接：

- 1.软链接，以路径的形式存在。类似于Windows操作系统中的快捷方式
- 2.软链接可以 跨文件系统 ，硬链接不可以
- 3.软链接可以对一个不存在的文件名进行链接
- 4.软链接可以对目录进行链接

硬链接：

- 1.硬链接，以文件副本的形式存在。但不占用实际空间。
- 2.不允许给目录创建硬链接
- 3.硬链接只有在同一个文件系统中才能创建

#### 命令参数

必要参数

- -b 删除，覆盖以前建立的链接
- -d 允许超级用户制作目录的硬链接
- -f 强制执行
- -i 交互模式，文件存在则提示用户是否覆盖
- -n 把符号链接视为一般目录
- -s 软链接(符号链接)
- -v 显示详细的处理过程

选择参数

- -S "-S<字尾备份字符串> "或 "--suffix=<字尾备份字符串>"
- -V "-V<备份方式>"或"--version-control=<备份方式>"
- --help 显示帮助信息
- --version 显示版本信息

e.g.

给文件创建软链接，为log2013.log文件创建软链接link2013，如果log2013.log丢失，link2013将失效：

```shell
ln -s log2013.log link2013
```

### Linux 挂载

1. 提一句Windows下，mount挂载，就是给磁盘分区提供一个盘符（C,D,E,...）。比如插入U盘后系统自动分配给了它I:盘符其实就是挂载，退优盘的时候进行安全弹出，其实就是卸载unmount。

2. Linux下，不像Windows可以有C,D,E,多个目录，Linux只有一个根目录/。在装系统时，我们分配给linux的所有区都在/下的某个位置，比如/home等等。

3. 提问者插入了新硬盘，分了新磁盘区sdb1。它现在还不属于/。

4. 我们虽然可以在一些图形桌面系统里找到他的位置，浏览管理里面的文件，但在命令行却不知怎么访问它的目录，比如无法使用cd或者ls。也无法在编程时指定一个目录对它操作。

5. 这时提问者使用了 mount /dev/sdb1 ~/Share/ ，把新硬盘的区sdb1挂载到工作目录的~/Share/文件夹下，之后访问这个~/Share/文件夹就相当于访问这个硬盘2的sdb1分区了。对/Share/的任何操作，都相当于对sdb1里文件的操作。

6. 所以Linux下，mount挂载的作用，就是将一个设备（通常是存储设备）挂接到一个已存在的目录上。访问这个目录就是访问该存储设备。

7. linux操作系统将所有的设备都看作文件，它将整个计算机的资源都整合成一个大的文件目录。我们要访问存储设备中的文件，必须将文件所在的分区挂载到一个已存在的目录上，然后通过访问这个目录来访问存储设备。挂载就是把设备放在一个目录下，让系统知道怎么管理这个设备里的文件，了解这个存储设备的可读写特性之类的过程。

8. 我们不是有/dev/sdb1 吗，直接对它操作不就行了？这不是它的目录吗？

9. 这不是它的目录。虽然/dev是个目录，但/dev/sdb1不是目录。可以发现ls/dev/sdb1无法执行。/dev/sdb1，是一个类似指针的东西，指向这个分区的原始数据块。mount前，系统并不知道这个数据块哪部分数据代表文件，如何对它们操作。

10. 插入CD，系统其实自动执行了 mount /dev/cdrom /media/cdrom。所以可以直接在/media/cdrom中对CD中的内容进行管理。

### vim

#### 查找

当你用 vim 打开一个文件后，因为文件太长，如何才能找到你所要查找的关键字呢？ 
在 vim 里可没有菜单-〉查找。

不过没关系，你在命令模式下敲斜杆( / )这时在状态栏（也就是屏幕左下脚）就出现了 “/” 然后输入你要查找的关键字敲回车就可以了。
如果你要继续查找此关键字，敲字符 n 就可以继续查找了。
敲字符N（大写N）就会向前查询。

使用 ? 也有相同的作用

#### vim 和 cat

cat 命令是linux系统下一个文本打印的命令，用于输出一个文本的信息到控制台上，该命令的输入类似于使用word打开一个文档，但是该文档不能编辑。

vi 命令是linux系统下用于文本查看、编辑的命令，不仅仅可以查看，还可以编辑。

### 路径

有/会认为是目录，没/会认为是文件。

加了/浏览器会指向一个目录，目录的话会读取默认文件index等等。没有/会先尝试读取文件，如果没有文件再找与该文件同名的目录，最后才读目录下的默认文件。

网址没有加上/会给服务器增加一个查找是否有同名文件的过程。

#### 展示当前所在目录

```shell
pwd
```



## echo

用于字符串的输出。命令格式：

```shell
echo string
```

您可以使用echo实现更复杂的输出格式控制。

### 1.显示普通字符串

```shell
echo "It is a test"
```

这里的双引号完全可以省略，以下命令与上面实例效果一致：

```shell
echo It is a test
```

### 2.显示转义字符

```shell
echo "\"It is a test\""
```

结果将是:

```shell
"It is a test"
```

同样，双引号也可以省略

### 3.显示变量

read 命令从标准输入中读取一行,并把输入行的每个字段的值指定给 shell 变量

```shell
#!/bin/sh
read name 
echo "$name It is a test"
```

以上代码保存为 test.sh，name 接收标准输入的变量，结果将是: 

```shell
[root@www ~]# sh test.sh
OK                     #标准输入
OK It is a test        #输出
```



## 日志

Session记录

```bash
/var/log/messages
```

ssh登录记录

```bash
last
```

### history

bash输入过的命令行记录

```bash
history
```

zsh 不必像bash一样需要配置~/.bashrc

```bash
vim /etc/profile
export HISTTIMEFORMAT='%F %T '
source ~/.bashrc
```

zsh 不必像bash一样需要配置，直接执行命令：

```zsh
history -E
    1   2.12.2013 14:19  cd ..

history -i
    1  2013-12-02 14:19  history -E

history -D
    1  0:00  history -E
```



## Shell

### 查看当前使用的shell

```bash
echo $SHELL
```

### 查看有哪些可用的shell

```bash
cat /etc/shells

# 结果
/bin/sh
/bin/bash
/usr/bin/sh
/usr/bin/bash
/bin/tcsh
/bin/csh
/bin/zsh
```

### zsh

#### 基本

MacOS 自带 zsh，Linux 需要进行安装 zsh。

#### 安装zsh

```bash
yum install zsh
```

#### 切换为zsh

```bash
chsh -s /bin/zsh
```

切换后需退出当前ssh连接，重新连接才可以生效。

`~/.zshrc` 中 `ZSH_THEME` 可以设置主题，例如：

```bash
ZSH_THEME="robbyrussell"
```

推荐使用的主题是 ys

```bash
ZSH_THEME="ys"
```

#### on-my-zsh

oh my zsh 项目提供了完善的插件体系，相关的文件在~/.oh-my-zsh/plugins目录下，默认提供了100多种，大家可以根据自己的实际学习和工作环境采用，想了解每个插件的功能，只要打开相关目录下的 zsh 文件看一下就知道了。插件也是在.zshrc里配置，找到plugins关键字，你就可以加载自己的插件了，系统默认加载 git ，你可以在后面追加内容，如下：

```text
plugins=(git textmate ruby autojump osx mvn gradle)
```

on-my-zsh 安装：

```bash
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

`~/.oh-my-zsh/themes` 中可以查看主题。

更详细的，例如安装插件可以参考：[终极 Shell——ZSH - 池建强 - 知乎](https://zhuanlan.zhihu.com/p/19556676)



## ssl

### 数字证书格式转换：.key和.crt转换成.pem格式

```bash
// .cer 转换成 .pem:
直接重命名后缀名即可
// .key 转换成 .pem:
openssl rsa -in temp.key -out temp.pem
// .crt 转换成 .pem:
openssl x509 -in tmp.crt -out tmp.pem
```



## 磁盘 Disk

### 查询磁盘容量

df 命令用于查看磁盘分区上的磁盘空间，包括使用了多少，还剩多少，默认单位是KB。

```bash
df -hl
```

执行结果如下：

![img](https://image-hosting.jellyfishmix.com/20210411155140.png)

执行的结果每列的含义：

- 第一列 Filesystem，磁盘分区

- 第二列 Size，磁盘分区的大小

- 第三列 Used，已使用的空间

- 第四列 Avail，可用的空间

- 第五列 Use%，已使用的百分比

- 第六列 Mounted on，挂载点

解释一下后面的h和l参数，h是把显示的单位改成容易辨认的单位，不再是默认的KB了，而l参数表示只显示本地磁盘分区，不包含的分区比如其他服务器共享的磁盘。

如果我们去掉l参数：

```bash
df -h
```

执行结果如下：
![img](https://image-hosting.jellyfishmix.com/20210411155701.png)

可以看到，和带着 l 参数的命令相比，执行的结果多了最下面一行，那是其他服务器的共享目录。

下面附上df命令的全部参数使用说明：

`-a或--all`：包含全部的文件系统；
`--block-size=<区块大小>`：以指定的区块大小来显示区块数目；
`-h或--human-readable`：以可读性较高的方式来显示信息；
`-H或--si`：与-h参数相同，但在计算时是以1000 Bytes为换算单位而非1024 Bytes；
`-i或--inodes`：显示inode的信息；
`-k或--kilobytes`：指定区块大小为1024字节；
`-l或--local`：仅显示本地端的文件系统；
`-m或--megabytes`：指定区块大小为1048576字节；
`--no-sync`：在取得磁盘使用信息前，不要执行sync指令，此为预设值；
`-P或--portability`：使用POSIX的输出格式；
`--sync`：在取得磁盘使用信息前，先执行sync指令；
`-t<文件系统类型>或--type=<文件系统类型>`：仅显示指定文件系统类型的磁盘信息；
`-T或--print-type`：显示文件系统的类型；
`-x<文件系统类型>或--exclude-type=<文件系统类型>`：不要显示指定文件系统类型的磁盘信息；
`--help`：显示帮助；
`--version`：显示版本信息。



## 常见 application

### nginx

nginx 配置文件路径查看

```
locate nginx.conf
```



## curl

### 下载文件
#### 利用 curl 下载文件。
使用内置 option：-o(小写)，自行指定下载后文件名称

```
curl -o dodo1.jpg http:www.linux.com/dodo1.JPG
```

使用内置 option：-O(大写)，以下载来源服务器上的名称保存文件到本地

```
curl -O http://www.linux.com/dodo1.JPG
```

#### 循环下载
有时候下载图片可以能是前面的部分名称是一样的，就最后的尾椎名不一样

```
curl -O http://www.linux.com/dodo[1-5].JPG
```

这样就会把 dodo1，dodo2，dodo3，dodo4，dodo5 全部保存下来

#### 下载重命名

```
curl -O http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

由于下载的 hello 与 bb 中的文件名都是 dodo1，dodo2，dodo3，dodo4，dodo5。因此第二次下载的会把第一次下载的覆盖，这样就需要对文件进行重命名。

```
curl -o #1_#2.JPG http://www.linux.com/{hello,bb}/dodo[1-5].JPG
```

这样在 hello/dodo1.JPG 的文件下载下来就会变成 hello_dodo1.JPG，其他文件依此类推，从而有效的避免了文件被覆盖

#### 分块下载

有时候下载的东西会比较大，这个时候我们可以分段下载。使用内置 option：-r

```
curl -r 0-100 -o dodo1_part1.JPG http://www.linux.com/dodo1.JPG
curl -r 100-200 -o dodo1_part2.JPG http://www.linux.com/dodo1.JPG
curl -r 200- -o dodo1_part3.JPG http://www.linux.com/dodo1.JPG
cat dodo1_part* > dodo1.JPG
```

这样就可以查看 dodo1.JPG 的内容了

#### 通过ftp下载文件

curl可以通过ftp下载文件，curl提供两种从ftp中下载的语法

```
curl -O -u 用户名:密码 ftp://www.linux.com/dodo1.JPG
curl -O ftp://用户名:密码@www.linux.com/dodo1.JPG
```

#### 显示下载进度条

```
curl -# -O http://www.linux.com/dodo1.JPG
```

#### 不会显示下载进度信息

```
curl -s -O http://www.linux.com/dodo1.JPG
```



## 引用/参考

[shell程序中 2> /dev/null 代表什么意思？ - 裕用ID的回答 - 知乎](https://www.zhihu.com/question/53295083/answer/135258024)

[linux标准输入输出 - 平面小狮子 - 简书](https://www.jianshu.com/p/d5ea4a8acfb9)

[Linux中查看磁盘大小、文件大小、排序方法小结 - lkforce - CSDN](https://blog.csdn.net/lkforce/article/details/80917306)

[Linux curl 命令下载文件 - jiapeng - 博客园](https://www.cnblogs.com/hujiapeng/p/8470099.html)

[终极 Shell——ZSH - 池建强 - 知乎](https://zhuanlan.zhihu.com/p/19556676)

[linux如何复制文件夹和移动文件夹](https://www.cnblogs.com/liaojie970/p/6746230.html)

[shell程序中 2> /dev/null 代表什么意思？ - 裕用ID的回答 - 知乎](https://www.zhihu.com/question/53295083/answer/135258024)

[Linux学习笔记（二）：什么是挂载？mount的用处在哪？ - 闻人翎悬 - CSDN](https://blog.csdn.net/qq_39521554/article/details/79501714)

[Linux 如何在 vi 里搜索关键字 - 蝈蝈俊 - CSDN](https://blog.csdn.net/ghj1976/article/details/6066069)

[linux查看某进程的开始时间、执行时长](https://blog.csdn.net/weixin_41712499/article/details/120055391)

