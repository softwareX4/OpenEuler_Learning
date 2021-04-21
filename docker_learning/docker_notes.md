# Docker学习笔记
[TOC]

## 容器技术
### 起源
- 重复搭建环境（开发、测试、运维）
- 只用于部署应用虚拟机太大（操作系统），且启动很慢
### 容器
- 相互隔离
- 长期反复使用
- 快速装载卸载
- 规格标准
#### 容器和虚拟机
虚拟机通过**操作系统**实现隔离，容器通过**隔离应用程序的运行时环境**（依赖的库和配置），但共享一个操作系统。

![](.img/vm.jpg)![](.img/docker.jpg)

容器轻量级且占用资源更少


## 概念
### 定义
Docker 是一个开源的、轻量级的容器引擎，主要运行于 Linux 和 Windows，用于创建、管理和编排容器。
- 镜像Image
包含有文件系统的、面向Docker引擎的、只读模板
提供运行环境
- 容器Container
轻量级的沙盒，极简的Linux系统环境(root权限、进程空间、用户空间、网络空间...)，以及运行在其中的应用程序。
Docker引擎利用容器来运行、隔离各个应用。容器是镜像创建的**应用实例**，可以**创建、启动、停止、删除**容器，各个容器之间是是相互隔离的，互不影响。
> 注意：镜像本身是只读的，容器从镜像启动时，Docker在镜像的上层创建一个**可写层**，镜像本身不变。
- 仓库Repository
类似于代码仓库，这里是镜像仓库，是Docker用来**集中存放镜像文件**的地方。注意与注册服务器（Registry）的区别：注册服务器是存放仓库的地方，一般会有多个仓库；而仓库是存放镜像的地方，一般每个仓库存放一类镜像，每个镜像利用tag进行区分，比如Ubuntu仓库存放有多个版本（12.04、14.04等）的Ubuntu镜像。

### docker工作过程
> 可以简单的把image理解为可执行程序，container就是运行起来的进程。
那么写程序需要源代码，那么“写”image就需要dockerfile，dockerfile就是image的源代码，docker就是"编译器"。
因此我们只需要在dockerfile中指定需要哪些程序、依赖什么样的配置，之后把dockerfile交给“编译器”docker进行“编译”，也就是**docker build**命令，生成的可执行程序就是**image**，之后就可以运行这个image了，这就是**docker run**命令，image运行起来后就是docker **container**。

- docker基于C/S架构，client处理用户输入命令，真正工作的是docker server(daemon)

1. **build** 
docker daemon根据dockerfile创建出image

![](.img/build.jpg)

2. **run** 
docker daemon加载image到内存，执行image，即container

![](.img/run.jpg)

3. **pull**  
docker daemon从registry下载写好的image到本地

![](.img/pull.jpg)

### 底层原理

![](.img/tech.png)

#### NameSpace
Linux中的PID、IPC、网络等资源是**全局**的，而NameSpace机制是一种**资源隔离方案**，在该机制下这些资源就不再是全局的了，而是属于某个特定的NameSpace，各个NameSpace下的资源互不干扰，这就使得每个NameSpace看上去就像一个独立的操作系统一样。

在linux下使用ps命令打印操作系统正在执行的进程，会有很多，比较特殊的是pid为1的init进程和pid为2的kthreadd进程，都由idle上帝进程创建出来。

![](.img/ps1.png)

- init
负责执行内核的一部分初始化工作和系统配置，也会创建一些类似 getty 的注册进程
- kthreaddd
负责管理和调度其他的内核进程。

![](.img/linuxProcess.jpg)

如果运行一个docker容器，再使用ps命令，并exec进入内部的bash，打印全部进程，则只有少数进程。
```sh
docker run -it -d centos
```

![](.img/run.png)

```sh
docker exec -it a6fa2404052f /bin/bash
```

![](.img/docker_exec.png)
 

**Docker 通过命名空间成功完成了与宿主机进程和网络的隔离。**
 
> [docker run 命令的 -i -t -d选项的作用](https://blog.csdn.net/claram/article/details/104228727)

#### Control groups

**Chroot**
在 Linux 系统中，系统默认的目录就都是以 / 也就是根目录开头的，chroot 的使用能够改变当前的系统根目录结构，通过改变当前系统的根目录，我们能够限制用户的权利，**在新的根目录下并不能够访问旧系统根目录的结构个文件**，也就建立了一个与原系统完全隔离的目录结构。

虽然有了NameSpace技术可以实现资源隔离，但**进程**还是可以不受控的访问**系统资源**（物理资源），比如CPU、内存、磁盘、网络等。如果其中的某一个容器正在执行 CPU 密集型的任务，那么就会影响其他容器中任务的性能与执行效率，导致多个容器**相互影响并且抢占资源**。

![](.img/cgroup.png)

为了控制容器中进程对资源的访问，Docker采用**control groups**技术(也就是cgroup)，有了cgroup就可以控制容器中进程对系统资源的消耗了，比如限制某个容器使用内存的上限、可以在哪些CPU上运行等等。

![](.img/cg1.png)

每一个 CGroup 都是一组被相同的标准和参数限制的进程，不同的 CGroup 之间是有层级关系的，也就是说它们之间可以从父类继承一些用于限制资源使用的标准和参数。

![](.img/cg2.png)

Linux 的 CGroup 能够为一组进程分配资源，也就是我们在上面提到的 CPU、内存、网络带宽等资源，通过对资源的分配，CGroup 能够提供以下的几种功能：

![](.img/cg3.png)

> 在 CGroup 中，所有的任务就是一个系统的一个进程，而 CGroup 就是一组按照某种标准划分的进程，在 CGroup 这种机制中，所有的资源控制都是以 CGroup 作为单位实现的，每一个进程都可以随时加入一个 CGroup 也可以随时退出一个 CGroup。

### 相关命令
#### Docker镜像
镜像是一个Docker的可执行文件，其中包括运行应用程序所需的所有代码内容、依赖库、环境变量和配置文件等。
**镜像管理**

![](.img/image.jpeg)

![](.img/image_mind.jpeg)

#### Docker容器和虚拟机的区别
- 相同点
  - 容器和虚拟机一样，都会对**物理硬件资源**进行共享使用。
  - 容器和虚拟机的**生命周期**比较相似（**创建、运行、暂停、关闭**等等）。
  - 容器中或虚拟机中都可以安装各种应用，如redis、mysql、nginx等。也就是说，在容器中的操作，如同在一个虚拟机(操作系统)中操作一样。
  - 同虚拟机一样，容器创建后，会存储在宿主机上：linux上位于/var/lib/docker/containers下
- 不同点
  - 虚拟机的创建、启动和关闭都是基于一个完整的操作系统。一个虚拟机就是一个完整的操作系统。而容器**直接运行在宿主机的内核**上，其本质上以一系列**进程的结合**。
  - 容器是轻量级的，虚拟机是重量级的。首先容器**不需要额外的资源**来管理(不需要Hypervisor、Guest OS)，虚拟机额外更多的性能消耗；其次创建、启动或关闭容器，如同创建、启动或者关闭**进程**那么轻松，而创建、启动、关闭一个操作系统就没那么方便了。
也因此，意味着在给定的硬件上能运行更多数量的容器，甚至可以直接把Docker运行在虚拟机上。

#### 容器的生命周期

![](.img/contain.png)

![](.img/contain_life.jpg)

[Docker生命周期相关命令](https://blog.csdn.net/wzf862187413/article/details/87803812)


#### Docker的容器与镜像关系
配置好容器后可以将其转化为镜像存储到仓库。
![](.img/repo.jpeg)


[Docker网络配置](http://blog.itpub.net/31556785/viewspace-2565390/)


