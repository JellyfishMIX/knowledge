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
sudo docker run -d --privileged --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
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

如果想要修改映射端口，请注意：

对于 rancher 的访问，必须要用https，即使用http访问，它还是会强制跳转到https。

经过多次试验，发现一个规律：

`localhost:8xxx` 会跳转到 `localhost:8443`，`localhost:9xxx` 会跳转到 `localhost:9443`，以此类推。

因此在使用 docker 启动 rancher 时，启动参数需要注意：

如果容器内的80端口映射到宿主机的 8xxx，那么容器内的443端口要映射到宿主机的 8443。

如果容器内的80端口映射到宿主机的 9xxx，那么容器内的443端口要映射到宿主机的 9443。

以此类推。

关于 rancher 修改端口映射，详情请见：[rancher 2 端口映射失败](https://github.com/JellyfishMIX/knowledge/blob/master/server/rancher/Rancher2端口映射失败.md)



## 引用/参考

[[docker]privileged参数 - 甘井子彭于晏 - CSDN](https://blog.csdn.net/weixin_43824748/article/details/109121287)