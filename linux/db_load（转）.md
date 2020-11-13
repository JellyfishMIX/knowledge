# db_load（转）



## 说明

db_load 将用户信息文件转换为数据库并使用hash加密。

e.g.

```bash
db_load -T -t hash -f /etc/vsftpd/vftpuser.txt /etc/vsftpd/vftpuser.db
```

e.g.

```bash
usage: db_load [-nTV] [-c name=value] [-f file]
        [-h home] [-P password] [-t btree | hash | recno | queue] db_file
usage: db_load -r lsn | fileid [-h home] [-P password] db_file
```

注：db_load命令需要安装 db4-utils这个软件包,在RHEL4.5中,这个软件包在第三个VCD光盘中。

参数：

- T
  选项-T允许应用程序能够将文本文件转译载入进数据库。由于我们之后是将虚拟用户的信息以文件方式存储在文件里的，为了让Vsftpd这个应用程序能够通过文本来载入用户数据，必须要使用这个选项。如果指定了选项-T，那么一定要追跟子选项-t。

- t
  子选项-t，追加在在-T选项后，用来指定转译载入的数据库类型。扩展介绍下，-t可以指定的数据类型有Btree、Hash、Queue和Recon数据库。

- f
  参数后面接包含用户名和密码的文本文件，文件的内容是：奇数行用户名、偶数行密码。



## 转自

[db_load - linux公社](http://linux.51yip.com/search/db_load)