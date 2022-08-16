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

快捷键：command + shift + .

### 解压

```bash
tar -zxvf ./xxx -C /xxx
```

tar是用来建立，还原备份文件的工具程序，它可以加入，解开备份文件内的文件。

-z	通过gzip指令处理备份文件

-x	从备份文件中还原文件

-v	显示指令执行过程

-f	指定备份文件

-C<目的目录>	切换到指定目录

### tail 动态实时展示文件的变化过程

tail命令可以动态实时展示文件的变化过程

example: 

```bash
tail -f error.2019-12-28.log
```

```bash
tail -f info.2019-12-28.log
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



## launchpad缺少图标

#### 步骤：

1. 打开Terminal终端机(可以在spotlight找）

2. 复制并贴上这句命令，按enter执行

```bash
mv ~/L*/Application\ Support/Dock/*.db ~/Desktop; killall Dock; exit
```

#### 注释：

1. 此命令会做一些刷新工作，不会删除已有资源和工作项目
2. 此命令会将桌面壁纸更改为默认壁纸，如果设置了非默认壁纸，再改回去就好



## 解决Chrome跨域问题

```bash
open -n /Applications/Google\ Chrome.app/ --args --disable-web-security --user-data-dir=/Users/qianshijie/MyChromeDevUserData/
```

完成调试后，需及时关闭并重启Chrome，重启后默认打开跨域保护。



## Macbook Pro 的 Touch Bar 中调节音量和亮度的键消失了的解决方案

打开 Terminal，运行`killall ControlStrip`，就可以了。



## 引用/参考

[Macbook Pro 的 Touch Bar 中调节音量和亮度的键消失了的解决方案 - -Hedon - CSDN](https://blog.csdn.net/Hedon954/article/details/106927258?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.edu_weight&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.edu_weight)