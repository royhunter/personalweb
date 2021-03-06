title: KVM安装与启动过程说明-Kernel源码编译方式
date: 2014-06-16 23:49:34
tags: Virtualization KVM QEMU
categories: KVM
---
《kvm安装与启动过程说明》进行了通用桌面系统的虚拟机安装。在本文中将介绍自行编译linux Kernel内核源码，无桌面OS虚拟机安装和启动。为后续内核调试打下基础。

<!--more-->

## 1. 安装QEMU-KVM

见前文。

## 2. 安装KVM内核模块

见前文。

## 3. 编译Linux源码

### 3.1	下载合适的内核源码

1. 链接：[http://www.kernel.org](http://www.kernel.org) 文件名linux-3.4.1.tar.gz
2. 解压下载的包：

```bash
$tar –zxvf linux-3.4.1.tar.gz
```

### 3.2	编译内核

```bash
$cd linux-3.4.1
$make defconfig
$make menuconfig
```

进入图形配置界面，若报错，可安装ncurses包

```bash
$sudo apt-get install libncurses5-dev
```

配置Virtualization项->

选择Kernel_based virtual Machine (KVM) support/
KVM for Intel processors support两项

配置kernel-hacking项->
选择compile kernel with debug info

![](http://7j1zng.com1.z0.glb.clouddn.com/1.png)

选择KVM支持与Intel 处理器。

![](http://7j1zng.com1.z0.glb.clouddn.com/2.png)

```bash
$make bzImage   //生成bzImage内核镜像
```

开始编译，编译完的结果如图：

主要的文件为vmlinux:  编译出来的最原始的内核文件，未压缩;

./arch/x86/boot/bzImage:  经过压缩后的img文件，比较大；

![](http://7j1zng.com1.z0.glb.clouddn.com/3.png)

## 4. BusyBox编译和安装

该工具用来制作initramfs镜像文件。

### 4.1	下载源码

1. 链接：[http://www.busybox.net](http://www.busybox.net)  文件为busybox-1.22.1.tar.gz2
2. 解压

```bash
$tar –xf busybox-1.22.1.tar.gz2
```

### 4.2	编译busybox

```bash
$make defconfig
$make menuconfig
```

因为Linux运行环境中是不带动态链接库的，所以必须以静态方式来编译BusyBox。

修改：

	Busybox Settings --->
		Build Options --->
			[*] Build BusyBox as a static binary(no shared libs)

![](http://7j1zng.com1.z0.glb.clouddn.com/4.png)

![](http://7j1zng.com1.z0.glb.clouddn.com/5.png)

![](http://7j1zng.com1.z0.glb.clouddn.com/6.png)

```bash
$make
$make install
```

### 4.3	制作initramfs

- 编写initrd启动脚本,$BUSYBOX为busybox包解压后的目录

```bash
$cd $BUSYBOX/_install
$mkdir proc sys dev etc etc/init.d
$vim $BUSYBOX/_install/etc/init.d/rcS
```

输入:

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

- 编写构建initrd镜像脚本

```bash
$vim $KERNEL/build-initrd.sh
```

输入：

```bash
#!/bin/sh
KERNEL = $(pwd)
BUSYBOX = $(find busybox* -maxdepth 0)
LINUX = $(find linux* -maxdepth 0)
cd $BUSYBOX/_install
find . | cpio -o --format=newrc > $KERNEL/rootfs.img
cd $KERNEL
gzip -c rootfs.img > rootfs.img.gz
```

- 运行脚本

```bash
$sudo ./build-initrd.sh
```

若有权限问题，更改下文件的权限，注意目录层次.
$KERNEL为linux源码解压后的目录，busybox源码也在该目录下。

- 成功后会在当前目录下生产rootfs.img.gz文件

## 5. Guest OS的安装运行

### 5.1	启动命令

```bash
$sudo /usr/local/kvm/bin/qemu-system-x86_64 -kernel \
	./kvm_guest/bzImage -initrd rootfs.img -append "root=/dev/ram rdinit=sbin/init noapic"
```

### 5.2	参数解释

![](http://7j1zng.com1.z0.glb.clouddn.com/18.png)

### 5.3	运行结果

![](http://7j1zng.com1.z0.glb.clouddn.com/17.png)

输入uname –a:

![](http://7j1zng.com1.z0.glb.clouddn.com/11.png)

输入cat /proc/cpuinfo:

![](http://7j1zng.com1.z0.glb.clouddn.com/12.png)

输入 cat /proc/meminfo:

![](http://7j1zng.com1.z0.glb.clouddn.com/13.png)

启动命令加入-smp 4选项后

```bash
$cat /proc/cpuinfo
```

![](http://7j1zng.com1.z0.glb.clouddn.com/14.png)