title: Container的内核技术基础：Namespace
date: 2015-12-07 21:49:00
tags: Linux namespace
categories: Docker
banner: http://7xoxkz.com1.z0.glb.clouddn.com/ns_2.png
---
容器的最重要的特性之一是资源的隔离性，Linux内核为Container的实现提供了比较重要的技术基础之一： Namespace技术。
Namespace这个词我们并不陌生，最常见的就是编程语言中的namespace概念，主要是命名空间或者package管理空间等。我的理解就是提供了一个新的环境或世界，该环境或世界提供了与其它环境或世界隔离，并拥有自我内部独立的运行规则。(漫威的漫画中经常有平行世界的概念，人物的发展分叉了，两个时空没有关系。。。扯远了）
<!--more--> 

## 什么是Namespace
------
参考[http://man7.org/linux/man-pages/man7/namespaces.7.html](http://man7.org/linux/man-pages/man7/namespaces.7.html)：

>  A namespace wraps a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. Changes to the global resource are visible to other processes that are members of the namespace, but are invisible to other processes. One use of namespaces is to implement containers.

Namespace是一种将全局系统资源的抽象成各自隔离的namespace中进程可见的一种实例化资源。我的理解就是资源进行实例化，申请者拥有各自的实例化对象，即拥有各自独立的内核系统。

Linux 支持6种类型的namespace：

```bash
Namespace      Constant        Isolates
  IPC         CLONE_NEWIPC    System V IPC, POSIX消息队列
  Network     CLONE_NEWNET    网络设备端口等
  Mount       CLONE_NEWNS     Mount points (挂载点)
  PID         CLONE_NEWPID    Process IDs (进程PID)
  User        CLONE_NEWUSER   User and group IDs (UID,GID)
  UTS         CLONE_NEWUTS    Hostname and NIS domain name
```

Namespace API:

	clone()     创建一个进程,可指定namespace的类型
	setns()     允许调用的进程加入一个指定的namespace
	unshare()   允许调用的进程脱离一个指定的namespace
	
这篇文章就CLONE_NEWPID（简单好理解一些）为主要例子来分析下Namespace的特性及其Linux内核的实现。

## Clone
------
从namespace的API:clone入手：

	int clone(int (*fn)(void *), void *child_stack,  int flags, void *arg, ...);

clone需要一个子进程的入口函数，一个栈空间，一些flag，和参数信息。

### 普通场景

	#define _GNU_SOURCE
	#include <sched.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/wait.h>
	#include <unistd.h>

	static char child_stack[100000];

	static int child_fn() {
	  printf("PID: %ld\n", (long)getpid());
	  printf("Parent PID: %ld\n", (long)getppid());
	  execv(container_args[0], container_args);
	  return 0;
	}

	int main() {
	  pid_t child_pid = clone(child_fn, child_stack+100000, SIGCHLD, NULL);
	  printf("clone() = %ld\n", (long)child_pid);
	 
	  waitpid(child_pid, NULL, 0);
	  printf("back to main()\n");
	  return 0;
	}
------
	root@royluo:~/work/container# gcc -o ns ns.c
	root@royluo:~/work/container# ./ns
	return child pid = 25984   <<<<<<<<<<<<clone返回的子进程PID
	PID: 25984   <<<<<<<<<<<子进程PID
	Parent PID: 25983    <<<<<<<<<<<父进程PID
	root@royluo:~/work/container# exit
	exit
	back to main()
	root@royluo:~/work/container#	
父进程调用clone创建了一个子进程，并输出了父子进程ID（25983， 25984)，这是没有创建新的namespace的情况。

### CLONE_NEWPID场景
	static int child_fn() {
	  printf("PID: %ld\n", (long)getpid());
	  printf("Parent PID: %ld\n", (long)getppid());
	   execv(container_args[0], container_args);
	  return 0;
	}

	int main() {
	  pid_t child_pid = clone(child_fn, child_stack+100000,
		CLONE_NEWPID | SIGCHLD, NULL);   //CLONE_NEWPID
	  printf("clone() = %ld\n", (long)child_pid);
	 
	  waitpid(child_pid, NULL, 0);
	  printf("back to main()\n");
	  return 0;
	}
------
	root@royluo:~/work/container# ./ns
	return child pid = 26013  <<<<<<<<<<<clone返回的子进程PID
	PID: 1   <<<<<<<<<<<  子进程PID
	Parent PID: 0   <<<<<<<<<<< 父进程PID
	root@royluo:~/work/container# exit
	exit
	back to main()
	root@royluo:~/work/container# 
但在系统ps能看到该子进程

	root@royluo:~# ps aux|grep 26013
	root     26013  0.0  0.4  21112  2012 pts/3    S+   07:25   0:00 /bin/bash
	root     26110  0.0  0.1  11740   932 pts/0    S+   07:26   0:00 grep --color=auto 26013
	root@royluo:~# 
显然CLONE_NEWPID在作怪。由于新建了PID的namespace，子进程就会在一个新的PID namespace里另起炉灶，成为了PID为1的进程，而父进程ID为0，表示不存在。
在Unix系统中，PID为1的进程很特殊，一般都是init进程，系统中其他进程都是由init继承而来，扮演终极父进程的角色。失去了父进程的子进程就都会以init作为它们的父进程。所以在新的namespace里由于PID的隔离性，导致了子进程从PID为1重新开始。其无法看到父进程的所有信息，但父进程却可以看到子进程。
下图将父子进程和其各自的namespace的关系陈述的很清楚，父进程在一个namespace，子进程在一个新的namespace，子进程所属父进程的namespace，而在子进程的namespace中是独自拥有的namespace。
![](http://7xoxkz.com1.z0.glb.clouddn.com/ns_1.PNG)

## 内核实现
------
在内核中每个进程都有一个struct task_struct 结构来表示，其中struct nsproxy 代表关联的namespace
```c
	struct task_struct {
		...
		/* open file information */
		struct files_struct *files;
		/* namespaces */
		struct nsproxy *nsproxy;   <<<<<<<<<<<<<<<namespace信息结构
		/* signal handlers */
		struct signal_struct *signal;
		struct sighand_struct *sighand;
		...
	}
	struct nsproxy {
		atomic_t count;
		struct uts_namespace *uts_ns;
		struct ipc_namespace *ipc_ns;
		struct mnt_namespace *mnt_ns;
		struct pid_namespace *pid_ns_for_children;  <<<<<<<<<<<PID namespace
		struct net 	     *net_ns;
	};
```
Clone是一个系统调用，在Linux内核中最终会调用do_fork():

	do_fork()
		copy_process()
			p = dup_task_struct(current); //创建子进程的task_struct，并复制父进程的一些空间信息
			copy_namespaces() //拷贝namespace信息
------	
	int copy_namespaces(unsigned long flags, struct task_struct *tsk)
	{
		//如果没有设上namespace相关的flag则直接返回，表示复用父进程的namespace
		if (likely(!(flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
			      CLONE_NEWPID | CLONE_NEWNET)))) {
			get_nsproxy(old_ns);
			return 0;
		}
		......
		//需要创建新的namespace
		new_ns = create_new_namespaces(flags, tsk, user_ns, tsk->fs);
		if (IS_ERR(new_ns))
			return  PTR_ERR(new_ns);
		tsk->nsproxy = new_ns;
		return 0;
	}
	static struct nsproxy *create_new_namespaces(unsigned long flags,
		struct task_struct *tsk, struct user_namespace *user_ns,
		struct fs_struct *new_fs)
	{
		new_nsp = create_nsproxy();
		...
		copy_mnt_ns() //创建新的Mount points Namespace
		...
		copy_utsname() //创建新的Hostname and NIS domain name Namespace
		...
		copy_ipcs()      //创建新的System V IPC, POSIX message queues Namespace
		...
		//重点关注创建新的Process IDs Namespace
		new_nsp->pid_ns_for_children =
		copy_pid_ns(flags, user_ns, tsk->nsproxy->pid_ns_for_children);
		...
		copy_net_ns()    //创建新的Network devices, stacks, ports, etc. Namespace
		...
	}
最终调用create_pid_namespace：

	static struct pid_namespace *create_pid_namespace(struct user_namespace *user_ns,
		struct pid_namespace *parent_pid_ns)
	{
		...
		//子进程的level总比父进程大1，可区分是否在一个namespace里
		unsigned int level = parent_pid_ns->level + 1; 
		...
		ns = kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL);
		ns->pidmap[0].page = kzalloc(PAGE_SIZE, GFP_KERNEL);
		...
		ns->level = level;
		ns->parent = get_pid_ns(parent_pid_ns);
		ns->user_ns = get_user_ns(user_ns);
		...
		//初始化PID Map表，这是PID另起炉灶的基础，该子进程第一个申请，也就占用了1号PID。
		set_bit(0, ns->pidmap[0].page);
		atomic_set(&ns->pidmap[0].nr_free, BITS_PER_PAGE - 1);

		for (i = 1; i < PIDMAP_ENTRIES; i++)
			atomic_set(&ns->pidmap[i].nr_free, BITS_PER_PAGE);
		...
	}
走到这里可以看到PID的分配空间已经和父进程隔离开来，甚至和父进程所在的PID Namespace隔离开来。该1号PID的进程的子进程的PID将在这个Namespace中”繁衍“。

REF:
1. [http://man7.org/linux/man-pages/man7/namespaces.7.html](http://man7.org/linux/man-pages/man7/namespaces.7.html)
2. [http://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces](http://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)
