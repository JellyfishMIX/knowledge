# Rancher 2 端口映射失败



## 发现问题

今天使用 docker 部署 rancher 的时候，端口映射失败了。
rancher 官方推荐的docker启动命令：

```
sudo docker run -d --privileged --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

参数含义：

-d: 在后台运行。

--privileged: 给予 container 中的 root 用户真正的 root 权限。

--restart=unless-stopped: docker启动后就会运行容器。

-p[宿主机端口:容器内端口]: 把容器内端口映射到宿主机端口上。

但是博主的 server 80和443端口都已经被占用，需要自行修改rancher的端口。

于是博主使用的命令是：

```
sudo docker run -d --privileged --restart=unless-stopped -p 8101:80 -p 8102:443 rancher/rancher
```

容器内80端口映射到宿主机的8101端口，容器内的443端口映射到宿主机的8102端口。

容器启动后，通过 `localhost:8101` 访问，发现页面被跳转到了 `localhost:8443`，并且访问失败，提示："This site can’t be reached"。

![image-20201027180014233](https://image-hosting.jellyfishmix.com/20201027180014.png)



## 解决问题

在网上搜寻相关资料，对于rancher 的访问，发现这样一句提示：

> 这里必须要用https。即使你用http访问，它还是会强制跳转到https。

这句话启发了博主，我通过8101端口访问，被自动跳转到了8443，和描述很像。

于是博主做了实验，修改了启动参数：

```
sudo docker run -d --privileged --restart=unless-stopped -p 8101:80 -p 8443:443 rancher/rancher
```

容器内80端口映射到宿主机的8101端口，容器内的443端口映射到宿主机的8443端口。

再次启动容器，访问 `localhost:8101`：

![image-20201027180515287](https://image-hosting.jellyfishmix.com/20201027180515.png)



页面跳转到了 `localhost:8443`，并且访问成功。



## 总结

对于 rancher 的访问，必须要用https，即使用http访问，它还是会强制跳转到https。

经过多次试验，发现一个规律：

`localhost:8xxx` 会跳转到 `localhost:8443`，`localhost:9xxx` 会跳转到 `localhost:9443`，以此类推。

因此在使用 docker 启动 rancher 时，启动参数需要注意：

如果容器内的80端口映射到宿主机的 8xxx，那么容器内的443端口要映射到宿主机的 8443。

如果容器内的80端口映射到宿主机的 9xxx，那么容器内的443端口要映射到宿主机的 9443。

以此类推。