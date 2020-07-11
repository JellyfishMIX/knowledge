## 电源

### 立即关机

```bash
sudo halt
```



## 存储

### 新建文件夹

```bash
mkdir xxx
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

### 查看内存占用情况

``top``

退出查看

``q``

``htop``

退出查看

``q``



## 进程

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



列出所有开放的端口

```bash
netstat -tulpn
```



### systemd

systemctl

/lib/systemd/



### 在后台持续运行进程

```bash
nohup command > path 2>&1 &
```

i.e.

```bash
nohup java -jar ./sell-eureka-server.jar > /dev/null 2>&1 &
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



## 引用/参考

[linux如何复制文件夹和移动文件夹](https://www.cnblogs.com/liaojie970/p/6746230.html)