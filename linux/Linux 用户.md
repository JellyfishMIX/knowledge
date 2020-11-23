# Linux 用户



## 常用操作

### userdel

userdel命令可以用于删除用户帐号及相关档案。 

语法：userdel [-r] 用户名 

参数：-r 用于彻底删除，用户HOME目录下的档案会被移除，在其他位置上的档案也将一一找出并删除，比如路径/var/mail/用户名 下的邮件。 

警告：userdel不允许你移除正在线上的使用者帐号。你必须kill此帐号现在在系统上执行的程序才能进行帐号删除。 

e.g.

彻底删除名为zhidao的用户： 

```bash
userdel -r zhidao
```

