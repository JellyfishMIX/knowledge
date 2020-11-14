# 在linux中添加ftp用户，并设置相应的权限



## 操作步骤

### 环境

ftp为vsftp。被限制用户名为test。被限制路径为/home/test

### 建用户

在root用户下：

```bash
# 增加用户test，并制定test用户的主目录为/home/test
useradd -d /home/test test
# 为test设置密码
passwd test
```

### 更改用户相应的权限设置：

```bash
# 限定用户test不能telnet，只能ftp
usermod -s /sbin/nologin test
# 用户test恢复正常
usermod -s /sbin/bash test
# 更改用户test的主目录为/test
usermod -d /test test
# 彻底删除 test 用户（删除用户及其相关文档）
userdel -r test
```

### 限制用户只能访问/home/test，不能访问其他路径

修改 /etc/vsftpd/vsftpd.conf 如下：

```bash
# 限制访问自身目录
chroot_list_enable=YES
# (default follows)
chroot_list_file=/etc/vsftpd/chroot_list
```

编辑 /etc/vsftpd/chroot_list 文件，将受限制的用户添加进去，每个用户名一行

**改完配置文件，不要忘记重启vsFTPd服务器**

```bash
systemctl start vsftpd.service
```

### 如果需要允许用户修改密码，但是又没有telnet登录系统的权限

```bash
# 用户telnet后将直接进入改密界面
usermod -s /usr/bin/passwd test
```



## 转自

[在linux中添加ftp用户，并设置相应的权限 - Reibin - 博客园](https://www.cnblogs.com/bienfantaisie/archive/2011/12/04/2275203.html)