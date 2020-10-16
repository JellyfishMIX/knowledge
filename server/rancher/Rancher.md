# Rancher



## Quick Start

rancher 官网：

```
https://rancher.com/quick-start/
```

rancher 中文网站：

```
https://www.rancher.cn
```

need docker

```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

请注意，此命令会导致 rancher container 一直重启

使用 `docker logs <container-id>` 查看一下日志：

```
ERROR: Rancher must be ran with the --privileged flag when running outside of Kubernetes
```

大意是：在k8s外运行rancher 需要 --privileged 启动参数。改成：

```
sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 80:443 rancher/rancher
```

--privileged 作用是给予 container 中的 root 用户真正的 root 权限。

>privileged 参数
>
>```
> $ docker help run 
> ...
> --privileged=false         Give extended privileges to this container
> ...
> ```
> 
> 大约在0.6版，privileged被引入docker。
> 使用该参数，container内的root拥有真正的root权限。
> 否则，container内的root只是外部的一个普通用户权限。
> privileged启动的容器，可以看到很多host上的设备，并且可以执行mount。
> 甚至允许你在docker容器中启动docker容器。



## 引用/参考

[[docker]privileged参数 - 甘井子彭于晏 - CSDN](https://blog.csdn.net/weixin_43824748/article/details/109121287)