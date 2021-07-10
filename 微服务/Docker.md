# Docker 容器技术

## Docker 简介

`基于 Go 语言的云开源项目。将”代码+环境“打包在一起，使应用达到跨平台无缝接轨使用。”一次封装，随处运行“`

Docker 解决了运行环境和配置问题的**软件容器**，方便做持续集成并有助于整体发布的容器虚拟化技术

### Docker 与虚拟机技术的区别

**虚拟机技术**

![image-20210318204046785](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318204046785.png)

从下至上：

- 基础设施。提供硬件设施和宿主机操作系统，它可以是个人电脑，服务器，或者是云主机。
- 虚拟机管理系统(Hypervisor)。可以在**主操作系统**上运行多个不同的**从操作系统**
  - Hypervisor 1：支持 MacOS 的 HyperKit，支持  Windows 的 Hyper-V、Xen 以及 KVM
  - Hypervisor 2：VirtualBox 和 VMWare workstation。
- 客户机操作系统(Guest OS)：Guest OS 的存在导致虚拟机占用过多的内存、磁盘、CPU 等资源。
- 依赖：每个 Guest OS 上都需要安装许多依赖。
- 应用。依赖满足之后，就可以在各个 Guest OS 上运行应用了，这些应用相互隔离。

缺点在于占用资源多，需要运行臃肿的 Guest OS，启动缓慢。

**Docker 容器技术**

![image-20210318205350234](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318205350234.png)

从下至上：

- 基础设施。硬件设施和主操作系统
- Docker 守护进程(Docker Daemon)：运行在操作系统上的后台进程，负责管理 Docker 容器。
- 容器内的应用依赖。所有的依赖打包在 Docker image 中，容器基于 image 创建。
- 应用。应用的源代码和依赖都打包在 Docker image 中，不同的应用需要不同的 Docker image，运行在不同的容器中，相互隔离。

**两者对比**

- 虚拟机技术需要虚拟出一套硬件，并在其上运行一个完整操作系统；
- 而 Docker 运行在宿主操作系统上，利用宿主机的内核，直接与操作系统通信，为各个 Docker 容器分配资源，将容器与主操作系统隔离，并且各个容器相互隔离。

## Docker 架构

![image-20210318210521030](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318210521030.png)

Docker 的基本组成：

- 镜像 Image：Docker 镜像就是一个**只读**的模板，镜像可以用来创建 Docker 容器。
- 容器 Container：利用镜像创建的运行实例，容器中运行着一个或者多组应用。
- 仓库 Repository：仓库是集中存放镜像文件的场所， Registry 是仓库服务器。

## Docker 镜像

镜像 Image 是一种轻量级, 可执行的独立软件包, **用来打包软件运行环境和基于运行环境开发的软件**. 它包含运行某个软件所需的所有内容, 包括代码, 运行时, 库, 环境变量和配置文件

### UnionFS

联合文件系统 (UnionFS) 是一种分层，轻量级且高性能的文件系统，支持对文件系统的修改作为一次提交来一层层的叠加，同时可以将不同目录挂载到同一个虚拟文件系统下。

UnionFS 是 Docker 镜像的基础。镜像可以通过分层来继承，基于基础镜像可以制作各种具体的应用镜像。

###  Docker 镜像加载原理

Docker 镜像实际上是一层一层的文件系统组成。

- bootfs(boot file system) 主要包含 bootloader 和 kernel
  - bootloader 主要引导加载 kernel，linux 刚启动时加载 bootfs 文件系统，在 Docker 镜像最底层为 bootfs。
  - 当 boot 加载完成之后，整个内核就存在内存中，此时内存的使用权已经由 bootfs 转交给内核，系统会卸载 bootfs
- rootfs(root file system)，在 bootfs 之上，包含 linux 系统中的 `/dev /proc /bin` 等标准目录和文件。

![image-20210318214205447](C:\Users\aasus\AppData\Roaming\Typora\typora-user-images\image-20210318214205447.png)

### 镜像特点

- 分层。下载镜像的过程中可以发现是一层一层在下载，宿主机上只要保存一份 base 镜像，其他所有镜像都可以使用这个 base 镜像。
- 只读。镜像都是只读的，容器启动时，一个新的可写层被加载到镜像的顶部（容器层）。

## Docker 数据卷

数据卷是一个可供一个或多个容器使用的特殊目录, 它绕过 UFS, 可以提供很多有用的特性:

- 数据卷可以在容器之间共享和重用
- 对数据卷的修改会立马生效
- 对数据卷的更新，不会影响镜像
- 卷会一直存在，直到没有容器使用

数据卷的使用，类似于 Linux 下对目录或文件进行 `mount`

### 数据卷容器

指定一个父容器挂载数据卷，其他的容器通过挂载这个父容器实现数据共享，这个父容器被称为**数据卷容器**

## Dockerfile

Dockerfile 使用来构建 Docker 镜像的构建文件，是由一系列命令和参数组成的脚本。以下是 centos 6.8 的 DockerFile。

```dockerfile
FROM scratch
MAINTAINER The CentOS Project <cloud-ops@centos.org>
ADD c68-docker.tar.xz /
LABEL name="CentOS Base Image" \
    vendor="CentOS" \
    license="GPLv2" \
    build-date="2016-06-02"

# Default command
CMD ["/bin/bash"]
```

每个关键字都必须为大写且后面要跟随至少一个参数，指令从上到下执行，每条指令都会创建一个新的镜像层，并对镜像进行提交。

### 执行流程

1. Docker 从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 执行类似 `docker commit` 的操作提交一个新的镜像层
4. Docker 再基于刚提交的镜像运行一个新容器
5. 执行 Dockerfile 中的下一条指令直到完成





