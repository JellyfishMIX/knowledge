# Docker



## 简介

可以粗糙地理解为轻量级的虚拟机。但它不是虚拟机。

![截屏2020-06-21下午4.44.58](https://image-hosting.jellyfishmix.com/20200621164704.png)

- 虚拟机（VM）
  - Hypervisor虚拟出了硬件层。
- Docker
  - 没有虚拟硬件层。
  - 使用Host OS中的namespace, control group等做到将application分离。因此Docker是轻量级的，程序启动速度提升，内存占用减小。



## Docker思想

- 集装箱
  
- 我们在不同机器上搬运程序的时候，有可能少发送了配置文件等，导致不能运行，程序散乱。按照标准放进集装箱，帮助标准化装配。
  
- 标准化

  - 运输方式

    把货物（程序）放进集装箱，把鲸鱼叫过来，让它把集装箱运到超级码头。使用的时候让鲸鱼把集装箱从超级码头搬运过来。

  - 存储方式

    程序copy到另一台机器的时候，需要指定一个目录，在去此目录中寻找。有了docker，可以不用关心应用存在何处，运行/停止时，只需要使用docker命令就好。

  - API接口

    docker提供了一系列控制API，包括启动停止等。如果用tomcat、nginx等服务器，启动停止等命令不同。接口标准化后，使用API即可控制所有命令。

- 隔离

  - 使用linux的内核限制技术（lxc），隔离进程和资源。轻量级的容器虚拟化技术，最大效率地隔离了进程和资源。比虚拟机更轻量。



## Docker的出现解决了什么问题

- 解决了系统环境不一致所带来的问题

  开发to运维：我本地运行没问题啊！怎么在你的服务器上跑不起来呢。（开始踢皮球）

  程序启动起来需要什么：操作系统，jdk，web server，代码，配置文件...

  docker来了，docker把操作系统，jdk，web server，代码，配置文件装进了集装箱。docker解决了系统环境不一致所带来的问题。

- 程序之间的隔离

  系统好卡，内存不够了，硬盘满了，服务变慢了，甚至敲终端都卡。

  使用docker解决这个问题。

  docker的隔离性解决了这个问题。

  如果一个程序发生了错误，异常吃性能吃资源，都不会导致其它程序发生错误。因为docker在程序启动的时候就为其设置了可使用的资源量，超过即杀掉。

- 机器临时扩展

  节日（类似于双11）时，负载迅速上涨，需要临时扩展机器，过完节日再把机器下线。

  这种临时扩展的需求，对于运维来说工作量很大，需要一台台机器去配环境。

  有了docker，只需要用鼠标点一点，即可从1台扩展到100台。因为有了docker标准化，在每台机器上使用标准的命令，即可将集装箱在docker中运行。



## Docker核心技术

- image 集装箱
- repository 超级码头
- container 运行程序的地方

docker去仓库把image拉到本地

### image

镜像可以理解为一个集装箱，本质是一系列文件，可以包括应用程序的文件，也包括应用运行环境的文件。

使用了linux存储技术unix fx（联合存储技术），是一种分层的文件系统，可以将不同的目录，挂载到同一个虚拟文件系统下。

![image-20201016172949546](https://image-hosting.jellyfishmix.com/20201016172949.png)

各层：

容器相关

自己添加的具体文件

具体的操作系统

最下层：操作系统的引导

### container

容器的本质是一个进程。container 和虚拟机的区别是：caontiner 的文件系统是一层一层的，并且下面的层都是 read only，只有最上面的一层是可写的。

container 在运行过程中如果要写入一个 image中的文件，因为 image 中的每一层都是read only的，所以在写文件前，会把此文件拷入最上层，再从最上层中对此文件进行修改。我们的应用读取一个文件时，会优先从顶层读取，如果顶层没有再去下层读取。

由于容器的最上层是可以修改的，而 image 是不可以修改的，这样可以保证同一个 image 可以生成多个 container 独立运行，container 之间不会互相影响。

### repository

存储各种 image 的超级码头。

#### famous repository

hub.docker.com

c.163.cn

### interal repository

对于比较私密的 image 需求，可以搭建私有仓库。



## 特点

- docker解决了软件包装的问题，缓和了开发和运维环境的差异，使开发和运维可以用相似的语言进行沟通。从系统环境开始，自底至上打包应用。
- 轻量级，对资源的有效隔离和管理。使用docker可以做到进程隔离，资源管理。
- docker可复用，可以通过image被重用，不需要从0开始构建，通过image来交付环境，用tag来指定image的版本，进行版本化。
- docker和持续集成、持续服务、持续部署、微服务等概念相辅相成。



## 架构

![截屏2020-06-21下午4.58.04](https://image-hosting.jellyfishmix.com/20200621165810.png)



## Docker command

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



## Docker command flow chart

![image-20201019181509114](https://image-hosting.jellyfishmix.com/20201019181509.png)

## Docker network

### 网络类型

Bridge模式

Host模式

None模式

### 端口映射

docker内的端口和host机器做映射。



## 引用/参考

[第一个docker化的java应用 - 刘果国 - 慕课网](https://www.imooc.com/learn/824)

