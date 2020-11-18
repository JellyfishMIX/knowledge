# vsftpd



## 常用操作

```undefined
VSFTP是一个基于GPL发布的类Unix系统上使用的FTP服务器软件，它的全称是Very Secure FTP 从此名称可以看出来，编制者的初衷是代码的安全。
```

在使用Vsftp服务是经常需要启动、停止、重启vsftp服务，下面是这几个操作使用的指令：

1、启动Vsftpd服务其命令为： service vsftpd start 或 /etc/init.d/vsftpd start

2、停止Vsftpd服务的命令为：service vsftpd stop 或 /etc/init.d/vsftpd stop

3、重新启动Vsftpd服务的命令为：service vsftpd restart 或 /etc/init.d/vsftpd restart

4、检查Vsftpd服务的运行状态：service vsftpd status



## completePendingCommad()

```java
* There are a few FTPClient methods that do not complete the
* entire sequence of FTP commands to complete a transaction.  These
* commands require some action by the programmer after the reception
* of a positive intermediate command.  After the programmer's code
* completes its actions, it must call this method to receive
* the completion reply from the server and verify the success of the
* entire transaction.
public boolean completePendingCommad() throws IOException; { return FTPReply.isPositiveCompletion(getReply()); } 
```

方法介绍中未说明，在何种情况下应该使用该方法。但是跟踪代码可以发现
这是一个同步阻塞方法，如果调用错误，会导致程序卡住假死在这里。

```java
// 卡住代码
String line = _controlInput_.readLine();
```

### 何时调用？

其实ftp功能，总结来说，只有上传和下载。只有在获取返回流时，才需要调用completePendingCommad方法，因为返回流不是立刻处理的。所以需用手动调用结束方法。

不可多加或者漏加，否则会导致程序卡死。



## 引用/参考

[Vsftpd服务重启、暂停命令 - 可可西里的星星 - 简书](https://www.jianshu.com/p/2d909f304e60)

[FTPClient中使用completePendingCommand方法注意事项 - 北海北_6dc3](https://www.jianshu.com/p/a90cc2aeefca)