title: Libvirt Introduce
date: 2015-11-19 21:34:46
tags: libvirt
categories: Virtualization
banner: http://7xogux.com1.z0.glb.clouddn.com/4.png
---
谈了很多KVM的实现细节，了解了如何使用qemu进行虚拟机的配置和创建，但是qemu提供的命令行参数过于复杂，不便于理解和操作。所以在技术上就有了更好的虚拟机管理工具libvirt。
libvirt是目前用的比较多的对KVM进行管理的虚拟化管理工具。其也支持其他虚拟化平台，并且提供了一整套API，提供了对虚拟机的管理，也提供了对虚拟化网络和存储的管理。
<!--more-->

## What's libvirt
Libvirt是目前用的比较多的对KVM进行管理的虚拟化管理工具。其也支持其他虚拟化平台，并且提供了一整套API，提供了对虚拟机的管理，也提供了对虚拟化网络和存储的管理。

Libvirt支持多种虚拟化方案，既支持包括KVM、QEMU、Xen、VMware、VirtualBox等在内的平台虚拟化方案，又支持OpenVZ、LXC等Linux容器虚拟化系统，还支持用户态Linux（UML）的虚拟化。

Libvirt是一个开源软件，使用的许可证是LGPL（Lesser General Public License）。

Reference: [http://libvirt.org/](http://libvirt.org/)

## Run Level
Libvirt的主要目标是为各种虚拟化工具提供一套方便、可靠的编程接口，用一种单一的方式管理多种不同的虚拟化提供方式.
从下图中看到，libvirt作为user-space管理工具与Hypervisor之间的适配层。让底层Hypervisor对上层用户空间的管理工具是可以做到完全透明的，因为libvirt屏蔽了底层各种Hypervisor的细节，为上层管理工具提供了一个统一的、较稳定的接口（API）。通过libvirt，一些用户空间管理工具可以管理各种不同的Hypervisor和上面运行的vm. 如下图所示：

![](http://7xogux.com1.z0.glb.clouddn.com/1.png)

## Glossary of terms

理解libvirt，就需要了解一些libvirt中经常用到的一些概念和术语：

### Domain

An instance of an operating system (or subsystem in the case of container  virtualization) running on a virtualized machine provided by the hypervisor.
一个操作系统的实例，可以理解为一个guest vm。

### Hypervisor

A layer of software allowing virtualization of a node in a set of virtual machines,  which may have different configurations to the node itself. 
VMM.

### Node

A single physical server. Nodes may be any one of many different types, and  are commonly referred to by their primary purpose. Examples are storage nodes, cluster nodes, and database nodes.
Host psysical mechine.

### Storage Pool

A collection of storage media, such as physical hard drives. A Storage Pool  is sub-divided into smaller containers called Volumes, which may then be allocated to one or more Domains.
存储池

### Volume

A collection of storage media, such as physical hard drives. A Storage Pool is sub-divided into smaller containers called Volumes, which may then be allocated to one or more Domains.   
存储卷.

## Architecture

### Object model

* Hypervisor connections
connection 是libvritAPI中主要的或者说比较顶层的对象。一个连接关联了一个特定的hypervisor，该hypervisor可能位于本机或
者是远程。其用virConnectPtr对象表示并用一个URI来标示。一个URI定义了连接hypervisor的路径。

* Guest domains
一个guest domain代表一个正在运行的虚拟机或者一份可以启动vm的配置，用virDomainPtr对象

* Virtual networks
虚拟网络提供连接一个或多个domain的网络设备的方法。使用virNetworkPtr对象标示。

* Storage pools
存储池对象提供了一种管理多种类型存储介质（比如本地磁盘，逻辑卷组......）的机制， virStoragePoolPtr

* Stroage volumes
virStorageVolPtr

* Host devices
virNodeDevPtr

### Driver model
libvirt对多种不同的Hypervisor的支持是通过一种基于驱动程序的架构来实现的。libvirt对不同的Hypervisor提供了不同的驱动.
Hypervisor drivers:
Xen:   Example local URI scheme xen:///.
QEMU:    Example privileged URI scheme qemu:///system.
UML:     Example privileged URI scheme uml:///system.
OpenVZ:   Example privileged URI scheme openvz:///system
LXC:    Example privileged
URI scheme lxc:///

![](http://7xogux.com1.z0.glb.clouddn.com/2.png)

## Libvirt Component
libvirt主要由三个部分组成，它们分别是：应用程序编程接口（API）库、一个守护进程（libvirtd）和一个默认命令行管理工具（virsh）.

1. 应用程序接口（API）是为了其他虚拟机管理工具（如virsh、virt-manager等）提供虚拟机管理的程序库支持。
2. libvirtd守护进程负责执行对节点上的域的管理工作，在用各种工具对虚拟机进行管理之时，这个守护进程一定要处于运行状态中，而且这个守护进程可以分为两种：一种是root权限的libvirtd，其权限较大，可以做所有支持的管理工作；一种是普通用户权限的libvirtd，只能做比较受限的管理工作。
3. virsh是libvirt项目中默认的对虚拟机管理的一个命令行工具

## Conclusion
以上主要大概了解了下libvirt的基本概念，后面将详细介绍libvirt的使用和实现细节。