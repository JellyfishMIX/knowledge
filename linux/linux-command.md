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



## 日志

Session记录

```bash
/var/log/messages
```

ssh登录记录

```bash
last
```

bash输入过的命令行记录

```bash
history
```



## Shell

### 查看当前使用的shell

```bash
echo $SHELL
```

### 查看有哪些可用的shell

```bash
cat /etc/shells
```

### zsh

#### 基本

MacOS自带zsh，Linux需要进行安装zsh。

#### 安装zsh

```bash
yum install zsh
```

#### 切换为zsh

```bash
chsh -s /bin/zsh
```

切换后需退出当前ssh连接，重新连接才可以生效。

`~/.zshrc`中`ZSH_THEME`可以设置主题，例如：

```bash
ZSH_THEME=”robbyrussell”
```

#### on-my-zsh

安装：

```bash
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

`~/.oh-my-zsh/themes`中可以查看主题。



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



## 引用/参考

[shell程序中 2> /dev/null 代表什么意思？ - 裕用ID的回答 - 知乎](https://www.zhihu.com/question/53295083/answer/135258024)

[linux标准输入输出 - 平面小狮子 - 简书](https://www.jianshu.com/p/d5ea4a8acfb9)

[Linux中查看磁盘大小、文件大小、排序方法小结 - lkforce - CSDN](https://blog.csdn.net/lkforce/article/details/80917306)