title: Docker Introduce
date: 2015-12-05 19:45:10
tags: Virtualization
categories: Docker
banner: http://7xoxkz.com1.z0.glb.clouddn.com/1.PNG
---
Docker最近这几年成为了虚拟化领域炙手可热的话题，发展也极为迅速，不同于KVM的OS级虚拟化解决方案，Docker从容器级虚拟化技术角度提供了一个开放的，及其轻量级的虚拟化平台。

> Docker helps you ship code faster, test faster, deploy faster, and shorten the cycle between writing code and running code.

<!--more--> 

## What is Docker?
在前面KVM的一系列的文章中，我们可以看到，KVM基于Linux内核资源，借助于硬件辅助虚拟化技术从OS级上提供了完整的虚拟化平台解决方案。而Docker从应用级别角度使用容器技术，隔离了容器的各自运行环境，成为了一个轻量级的虚拟化平台。
Docker提供了一种能力，使任何应用程序能够安全地隔离于其他应用在一个独立的容器中运行。 隔离与安全性可以使多个容器可以共同运行于Host。其轻量级的特性规避了hypervisor的运行消耗，从而可以充分利用硬件资源。

![](http://7xoxkz.com1.z0.glb.clouddn.com/2.PNG)

在上图Virtual Machine与container平台对比中可以看到，Virtual Machine平台主要是通过Host上的hypervisor运行的多个Guest OS来达到运行环境的隔离性，而Container技术主要通过Container引擎来驱动隔离性，没有引入新的OS，减轻了系统的运行负担，使得上层应用程序可以直接复用Host OS的资源和功能，达到轻量级的运行特性。

## Docker的优势
1. 更快的交付你的应用
Docker可以让一个团队工作成员各自工作在本地的容器中，并通过持续的集成和迭代加快开发速度，提高开发效率。这一特性很像代码版本的分布式控制系统，用过GIT进行代码管理的开发者应该对此更理解一些。

2. 更容易的部署和扩展
Docker的容器封闭性可以更容易的部署在任意一台Linux系统上，而这Linux系统可以是一台真实的物理机，或者虚拟机，甚至是云平台。轻量级的特性支持容器的规模的扩展，充分利用每一台Physical Machine的Resource。

3. 运行的效率
Docker支持应用程序直接运行于隔离的环境，复用host os的资源和功能，省去了hypervisor对Guest OS的繁琐的管理和运行开销，使应用更能达到一种实时运行的状态。

## Docker的架构
Docker使用了CS架构，Docker Client与Docker Daemon进行交互，Daemon主要进行image的building,running和distributing.
Docker Client和Docker Daemon可以运行于同一系统中，或者Client与远程的Daemon进行通信，通信的接口使用socket或者RESTful API。

在我本机上运行的是Ubuntu14.04，集成了Docker的安装包，所以可以直接安装：
```bash
root@royluo:~# apt-get install -y docker.io
```

安装完启动后，可以看到系统中运行了docker daemon：
```bash
root@royluo:~# ps aux|grep docker
root     13971  1.5  1.8 235384  9156 ?        Ssl  07:53   0:00 /usr/bin/docker -d
```
由于Docker Client与Daemon共用的是同一个可执行文件，所以Daemon启动时加上-d选项。而Client直接使用docker命令。
```bash
root@royluo:~# docker --help
Usage: docker [OPTIONS] COMMAND [arg...]

A self-sufficient runtime for linux containers.
......
```

摘自官网的一幅图：可以看到Client和Daemon的交互关系.
![](http://7xoxkz.com1.z0.glb.clouddn.com/3.PNG)

其中也说明了Docker的内部组成：
### Docker images.
一个Docker image是一个只读的模板，其包含了需要运行的应用所需的组件，例如库，可执行程序等，是一个静态的概念。

### Docker registries.
我的理解是image的仓库，提供类似于Github的服务，其提供image的管理操作。

### Docker containers.
Container就是一个运行的环境，可以create，run，start，stop，move，delete...

## Docker背后的技术
Docker是由GO语言开发，利用了Linux内核的一些特性：

### Namespaces
Namespaces技术提供了container运行的隔离环境。

### Control groups
Cgroup技术提供了container运行的资源限制措施。

### Union file systems
UFS提供了一种能够创建层级文件系统的功能。

这些技术后面会一步步进行学习和分析。

Ref: [https://docs.docker.com/engine/introduction/understanding-docker/](https://docs.docker.com/engine/introduction/understanding-docker/)
