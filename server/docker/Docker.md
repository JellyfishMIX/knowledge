# Docker



## 简介

可以粗糙地理解为轻量级的虚拟机。但它不是虚拟机。

![截屏2020-06-21下午4.44.58](https://image-hosting.jellyfishmix.com/20200621164704.png)

- 虚拟机（VM）
  - Hypervisor虚拟出了硬件层。
- Docker
  - 没有虚拟硬件层。
  - 使用Host OS中的namespace, control group等做到将application分离。因此Docker是轻量级的，程序启动速度提升，内存占用减小。



## 特点

- docker解决了软件包装的问题，缓和了开发和运维环境的差异，使开发和运维可以用相似的语言进行沟通。从系统环境开始，自底至上打包应用。
- 轻量级，对资源的有效隔离和管理。使用docker可以做到进程隔离，资源管理。
- docker可复用，可以通过image被重用，不需要从0开始构建，通过image来交付环境，用tag来指定image的版本，进行版本化。
- docker和持续集成、持续服务、持续部署、微服务等概念相辅相成。



## 架构

![截屏2020-06-21下午4.58.04](https://image-hosting.jellyfishmix.com/20200621165810.png)



