title: runC
date: 2016-01-02 00:55:55
tags: Container
categories: Container
banner: http://7xoxkz.com1.z0.glb.clouddn.com/runc.png
---
最近一直想研究下libcontainer的源码，看下是其是如何提供Linux内核的关于Namespace和Cgroup的API封装的，无意看到在2015年6月份OCI发布的一个开发容器标准和一个开源项目RUNC，感觉有点意思，准备从其入手，了解一些想了解的东西。
<!--more--> 

## What's runc 
Linux基金会于2015年6月成立了OCI（Open Container Initiative）组织，从容器格式和运行环境方面定义了一个开放式的标准，其目标是为了让container的运行不依赖于任何的操作系统，硬件和特定的容器运行工具等.
runc is a CLI tool for spawning and running containers according to the OCF specification.
官网：[http://runc.io/](http://runc.io/)
从其代码目录组织上来看，其主要是通过libcontainer来达到对容器的创建和控制。
[https://github.com/opencontainers/runc](https://github.com/opencontainers/runc)

下面根据官网的一些介绍，运行了个小小的Example，体验了下runc, opencontainer。

## 安装GO
runc和libcontainer都是使用go语言编程，所以需要go语言的开发环境，这里对GO的安装不做详细介绍，请自行到官网根据install guide进行安装。
[https://golang.org/doc/install](https://golang.org/doc/install)

## 安装runc
```bash
#mkdir -p ${GOPATH}/src/github.com/opencontainers
#cd ${GOPATH}/src/github.com/opencontainers
#git clone https://github.com/opencontainers/runc
#cd runc
#make
#make install
```
如果出现下面错误：
```bash
Godeps/_workspace/src/github.com/seccomp/libseccomp-golang/seccomp.go:25:22: fatal error: seccomp.h: No such file or directory
 // #include <seccomp.h>
 ```
 那就是缺少libseccomp， 使用apt-get install libseccomp-dev即可安装通过。
验证下runc是否成功安装
 ```bash
root@beijing:/home/beijing/work# which runc
/usr/local/bin/runc
```

## 使用runc
一个标准的容器包由三个部分组成：1. config.json; 2. runtime.json; 3. rootfs
### runc spec
根据OCI容器标准，运行OFC需要两个符合标准格式的配置文件，文件格式为json形式：
1. config.json
2. runtime.json
我们可以自己按照标准来写这两个配置文件，或者使用runc的自带功能自动生成：
```bash
#runc spec
```

### rootfs
还需要造一个rootfs，按照官网的说法，可以使用docker image中的busybox手动剥离一个rootfs为我们所用，此处所用docker并不是说runc依赖docker，而是使用docker来做一个rootfs.
```bash
#docker pull busybox
#docker export $(docker create busybox) > busybox.tar
#mkdir rootfs
#tar -C rootfs -xf busybox.tar
```
这样就比较方便的形成了一个rootfs：
```bash
root@beijing:/home/beijing/work/runc# ls -l rootfs
total 48
drwxr-xr-x 2 root root 12288 Dec  9 00:44 bin
drwxr-xr-x 4 root root  4096 Jan  1 23:23 dev
drwxr-xr-x 2 root root  4096 Jan  1 23:23 etc
drwxr-xr-x 2   99   99  4096 Dec  9 00:44 home
drwxr-xr-x 2 root root  4096 Jan  1 23:23 proc
drwxr-xr-x 2 root root  4096 Dec  9 00:44 root
drwxr-xr-x 2 root root  4096 Jan  1 23:23 sys
drwxrwxrwt 2 root root  4096 Dec  9 00:44 tmp
drwxr-xr-x 3 root root  4096 Dec  9 00:44 usr
drwxr-xr-x 4 root root  4096 Dec  9 00:44 var
root@beijing:/home/beijing/work/runc# 
```

### runc start
有了这三样东西，我们就可以运行一个符合开放式标准的容器：
```bash
root@beijing:/home/beijing/work/runc# runc start
/ # ps
PID   USER     TIME   COMMAND
    1 root       0:00 sh
    6 root       0:00 ps
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
/ # 
```

## runc command usage
runc作为一个container运行的命令行工具，支持的功能还不少：
```bash
COMMANDS:
   start        create and run a container
   checkpoint   checkpoint a running container
   events       display container events such as OOM notifications, cpu, memory, IO and network stats
   restore      restore a container from a previous checkpoint
   kill         kill sends the specified signal (default: SIGTERM) to the container's init process
   spec         create a new specification file
   pause        pause suspends all processes inside the container
   resume       resume resumes all processes that have been previously paused
   exec         execute new process inside the container
   help, h      Shows a list of commands or help for one command
```

## conclusion
总体而言，使用runc run一个container是比较简单的，其确实是一个轻量级的命令行工具，并且没有类似docker daemon那样的后台程序进行支持。
