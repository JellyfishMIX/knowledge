# docker command



## 常用操作

```bash
// 运行镜像（images）
docker run xxx(name)
// 停止镜像
docker stop xxx(IMAGE ID)
// 查看本地镜像（images）
docker images
// 删除本地镜像（images）
docker rmi xxx(IMAGE ID)
// 查看container（正在运行的）
docker ps
// 查看container（所有的）
docker ps -a
// 删除container
docker rm xxxx(container ID)
// 拷贝
docker cp index.xml xxx(IMAGE ID)://usr/share/nginx.html
```



| 命令          | 功能                          |
| ------------- | ----------------------------- |
| docker pull   | 获取image                     |
| docker build  | 创建image                     |
| docker images | 列出image                     |
| docker run    | 运行container                 |
| docker ps     | 列出container                 |
| docker rm     | 删除container                 |
| docker rmi    | 删除image                     |
| docker cp     | 在host和container之间拷贝文件 |
| docker commit | 保存改动为新的image           |
| docker start  | 启动一个已经停止的container   |



## 操作细节

#### docker run

![docker run 和 docker start的区别](https://image-hosting.jellyfishmix.com/20200718112835.png)

`docker run`只有在第一次运行时使用，将镜像放到容器中，以后再次启动这个容器的时候，只需要使用命令`docker start`就可以。
docker run相当于执行了两步操作：将镜像（Image）放到容器（Container）中，这一步过程叫做`docker create`，然后将容器启动，使之变成运行时容器（docker start）。

```bash
docker run -p 8080:80 -d daocloud.io/nginx
```

运行daocloud.io/nginx镜像。

##### 参数

- -p 把docker nginx这个image的80端口映射到本地8080端口。
- -d 允许nginx这个程序直接返回。把nginx所在的container作为指挥进程来执行。

##### 使用docker run命令来启动容器，docker在后台运行的标准操作包括：

1. 检查本地是否存在指定的镜像，不存在则从公有仓库下载
2. 使用镜像创建并启动容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读可写层
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口道容器中去
5. 从地址池分配一个ip地址给容器
6. 执行用户指定的应用程序
7. 执行完毕之后容器被终止



### docker start

`docker start`的作用是：重新启动已经存在的容器。也就是说，如果使用这个命令，我们必须先要知道这个容器的ID、或者这个容器的名字，我们可以使用`docker ps -a`命令找到这个容器的信息。



### docker cp

```bash
docker cp index.xml xxx(IMAGE ID)://usr/share/nginx.html
```

把本地OS当前目录的`index.xml`文件拷贝到`xxx(IMAGE ID)://usr/share/nginx.html`位置。

docker在容器内所做的改动都是暂时的。stop容器后，容器所发生的改动不会被保存下来。

如果想保存改动，需要执行：

```bash
docker commit -m 'some comments' xxxx(container ID) xxxx(name)
```

此操作会生成一个新的镜像。



## Docker command example

1.

```
docker pull [OPTIONS] NAME[:TAG]
```

从仓库拉取一个 image。

[OPTIONS] 指定拉取时的参数。

[:TAG]指定拉取的版本，如果不填默认为 latest。

2.

```
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

查看本机有哪些 images。

[OPTIONS] 指定命令的参数。

[REPOSITORY[:TAG]] 可以指定仓库:版本的 image。

3.

```
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```

[OPTIONS] 指定命令的参数。

[OPTIONS]加 `-d`：后台运行。

[OPTIONS]加 `-p`：映射到 host 机器的某个端口。[host port]:[docker interal port] e.g.: `-p 8080:80`。

[OPTIONS]加 `-P`：把所有docker interal的端口，一对一映射到host机器的随机端口。

[:TAG] 指定的版本。

4.

```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

Run a command in a running container.

Commonly used parameters:

-i: 保证输入有效。

-t: 分配一个伪终端。

e.g.

使用bash进入一个 container

```
docker exec -it [container_id] /bin/bash
```

5.

```
docker build -t [NAME]:[TAG] [Dockerfile path]
```

构建一个docker image

-t：指定一个 [NAME]:[TAG] 给构建后的 image。

[Dockerfile path]： the directory where the Dockerfile is located.

6.

```
docker inspect image
```

see image details

1.

```
docker pull [OPTIONS] NAME[:TAG]
```

从仓库拉取一个 image。

[OPTIONS] 指定拉取时的参数。

[:TAG]指定拉取的版本，如果不填默认为 latest。

2.

```
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

查看本机有哪些 images。

[OPTIONS] 指定命令的参数。

[REPOSITORY[:TAG]] 可以指定仓库:版本的 image。

3.

```
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
```

[OPTIONS] 指定命令的参数。

[OPTIONS]加 `-d`：后台运行。

[OPTIONS]加 `-p`：映射到host机器的某个端口。[host port]:[docker interal port]e.g.: `-p 8080:80`。

[OPTIONS]加 `-P`：把所有docker interal的端口，一对一映射到host机器的随机端口。

[:TAG] 指定的版本。

4.

```
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

Run a command in a running container.

Commonly used parameters:

-i: 保证输入有效。

-t: 分配一个伪终端。

e.g.

使用bash进入一个 container

```
docker exec -it [container_id] /bin/bash
```

5.

```
docker build -t [NAME]:[TAG] [Dockerfile path]
```

构建一个docker image

-t：指定一个 [NAME]:[TAG] 给构建后的 image。

[Dockerfile path]： the directory where the Dockerfile is located.

6.

```
docker inspect image
```

see image details



## Dockerfile

可以在项目根目录中，创建一个 docker.sh 自动化运行一系列 docker 构建、推送至仓库等命令。

如果报错:

Permission denied

说明没有权限。

解决办法： 

修改该文件 docker.sh 的权限:

chmod 777 aa.sh