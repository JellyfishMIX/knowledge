# Linux log



## SSH登录情况分析

### wtmp日志

```
last
last -x -F
```

### 查看在线用户情况

（1）w 命令用于显示已经登陆系统的用户列表，并显示用户正在执行的指令。单独执行w命令会显示所有的用户，您也可指定用户名称，仅显示某位用户的相关信息。

（2）who am i 显示你的出口IP地址，该地址用于SSH连接的源IP

```bash
who am i
root     pts/0        2018-03-29 04:12 (111.204.243.8)
```

### SSH登录日志分析

```bash
cat /var/log/secure |more
less /var/log/secure|grep 'Accepted'  
less /var/log/auth.log|grep 'Accepted'
```

检查/var/log目录下的secure（CentOS）或者auth.log（Ubuntu），如果存在大量异常IP高频率尝试登录，且有成功登录记录（重点查找事发时间段），在微步在线上查询该登录IP信息，如果为恶意IP且与用户常用IP无关，则很有可能为用户弱口令被成功爆破。

/var/log/其他日志说明：

```bash
/var/log/message  一般信息和系统信息
/var/log/secure  登陆信息
/var/log/maillog  mail记录
/var/log/utmp 
/var/log/wtmp登陆记录信息（last命令即读取此日志）
```



## 引用/参考

[Linux入侵分析（二）分析SSH登录日志 - Mike_Rock_Cloud - 51CTO博客](https://blog.51cto.com/winhe/2114533)