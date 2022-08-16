# nginx



## 查询 nginx 配置文件

```bash
locate nginx.conf
```

或

```bash
ps aux|grep nginx
# result
root     12800  0.0  0.0 112684   736 pts/2    S+   12:20   0:00 grep --color=auto nginx
root     26770  0.0  0.0  56316  1060 ?        Ss   Apr24   0:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
root     26772  0.0  0.7  68048 14312 ?        S    Apr24   1:47 nginx: worker process
root     26773  0.0  0.8  68856 15156 ?        S    Apr24   1:58 nginx: worker process
```

更详细的：[nginx快速查看配置文件的方法 -  傲雪星枫 - CSDN](https://blog.csdn.net/fdipzone/article/details/77199042)



## 查看当前 nginx 是否正在运行

```bash
# 查看正在被监听的端口
netstat -ntlp
netstat -ntlp | grep nginx
```



## 启动 nginx

```bash
cd usr/local/nginx/sbin
注意：usr/local/nginx 是安装目录
./nginx
```



## 停止 nginx

```bash
cd usr/local/nginx/sbin
注意：usr/local/nginx 是安装目录
./nginx -s stop
```

