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
- `&` 表示在后台运行。

e.g.

```bash
nohup java -jar ./sell-eureka-server.jar > /dev/null 2>&1 &
```

- `/dev/null` 



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



## 引用/参考

