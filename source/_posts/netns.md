title: Network Namespace
date: 2015-12-13 21:05:09
tags: Linux namespace
categories: Docker
banner: http://7xoxkz.com1.z0.glb.clouddn.com/4.png
---
周末学习了针对网络相关的Linux Namespace模块，搜罗了一些资料，搭建了小小的测试环境，查阅了一些源码实现，此处做了下小小的笔记。
<!--more--> 

## Network Namespace
Network Namespace是Linux Kernel实现的六个Namespace之一，其实现了对内核网络模块的隔离。一个network namespace提供了一个隔离的空间，该隔离范畴包括：
1. 网络设备接口；
2. IPv4，IPv6协议栈；
3. IP路由表；
4. 防火墙规则；
5. /proc/net and /sys/class/net目录树； 
6. socket 等 

ip命令行工具提供了netns的操作，包括创建，删除network namespace等操作，可以以此分析下netns的实现原理。

## 简单的netns通信实例
这里主要使用ip命令行工具来搭建一个比较简单的不同namespace之间通信的拓扑图。如下：
![](http://7xoxkz.com1.z0.glb.clouddn.com/ns_3.PNG)
1. 创建两个network namespace：beijing，shanghai
```bash
root@royluo-VirtualBox:/home/royluo# ip netns add beijing
root@royluo-VirtualBox:/home/royluo# ip netns add shanghai
root@royluo-VirtualBox:/home/royluo# ip netns
shanghai
beijing
root@royluo-VirtualBox:/home/royluo# ls /var/run/netns
beijing  shanghai 
```
2. 创建一个bridge：br0
```bash
root@royluo-ThinkPad-T430:~# brctl addbr br0
root@royluo-ThinkPad-T430:~# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.000000000000       no
root@royluo-ThinkPad-T430:/home/royluo# ip link set br0 up
```
3. 创建两个veth interface，并分别设置对应的peer name，将两个veth加入br0
```bash
root@royluo-VirtualBox:/home/royluo# ip link add eth0-b type veth peer name veth-b
root@royluo-VirtualBox:/home/royluo# ip link add eth0-s type veth peer name veth-s
root@royluo-VirtualBox:/home/royluo# ip link
......
3: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default 
    link/ether fe:60:98:7f:e8:19 brd ff:ff:ff:ff:ff:ff
4: veth-b: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 06:72:2f:b2:09:7a brd ff:ff:ff:ff:ff:ff
5: eth0-b: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
    link/ether 56:47:69:cf:8b:76 brd ff:ff:ff:ff:ff:ff
6: veth-s: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 36:7a:e4:4b:09:26 brd ff:ff:ff:ff:ff:ff
7: eth0-s: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:af:32:75:f8:e6 brd ff:ff:ff:ff:ff:ff
root@royluo-VirtualBox:/home/royluo# brctl addif br0 veth-b
root@royluo-VirtualBox:/home/royluo# brctl addif br0 veth-s
root@royluo-VirtualBox:/home/royluo# brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.06722fb2097a       no              veth-b
                                                        veth-s
root@royluo-VirtualBox:/home/royluo# ip link set veth-b up
root@royluo-VirtualBox:/home/royluo# ip link set veth-s up
```
4. 将veth的另一端分别加入beijing，shanghai的namespace中,在各自的ns中只能看到自己拥有的dev；
```bash
root@royluo-VirtualBox:/home/royluo# ip link set eth0-b netns beijing
root@royluo-VirtualBox:/home/royluo# ip link set eth0-s netns shanghai

root@royluo-VirtualBox:/home/royluo# ip netns exec beijing ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
5: eth0-b: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 56:47:69:cf:8b:76 brd ff:ff:ff:ff:ff:ff

root@royluo-VirtualBox:/home/royluo# ip netns exec shanghai ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
7: eth0-s: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 5e:af:32:75:f8:e6 brd ff:ff:ff:ff:ff:ff
```
5. 将各自namespace中的接口up，同时设置ip地址；
```bash
root@royluo-VirtualBox:/home/royluo# ip netns exec beijing ip link set dev lo up 
root@royluo-VirtualBox:/home/royluo# ip netns exec beijing ip link set dev eth0-b up
root@royluo-VirtualBox:/home/royluo# ip netns exec beijing ip address add 10.0.0.1/24 dev eth0-b

root@royluo-VirtualBox:/home/royluo# ip netns exec shanghai ip link set dev lo up
root@royluo-VirtualBox:/home/royluo# ip netns exec shanghai ip link set dev eth0-s up 
root@royluo-VirtualBox:/home/royluo# ip netns exec shanghai ip address add 10.0.0.2/24 dev eth0-s
```
6. 互ping对方，此时通过bridge完成通信的目标达成；
```bash
root@royluo-VirtualBox:/home/royluo# ip netns exec beijing ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.110 ms
^C
--- 10.0.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.050/0.080/0.110/0.030 ms
root@royluo-VirtualBox:/home/royluo# ip netns exec shanghai ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.052 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.054 ms
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.052/0.053/0.054/0.001 ms
```

## 源码分析
ip命令行工具的源码下载：[https://github.com/shemminger/iproute2](https://github.com/shemminger/iproute2),其作为用户态工具主要通过netlink机制与内核进行交互。
### ip netns add xxx
此功能主要是创建一个新的network namespace；主要代码在ipnetns.c
```bash
static int netns_add(int argc, char **argv)
{
	//创建/var/run/netns
	mkdir(NETNS_RUN_DIR, S_IRWXU|S_IRGRP|S_IXGRP|S_IROTH|S_IXOTH);
	//创建打开/var/run/netns/xxx
	fd = open(netns_path, O_RDONLY|O_CREAT|O_EXCL, 0);
	//脱离父进程的netns，此时拥有了独立的netns
	unshare(CLONE_NEWNET)  <<<<<<unshare() API
	//将/proc/self/ns/net mount到/var/run/netns/xxx
	mount("/proc/self/ns/net", netns_path, "none", MS_BIND, NULL)
}
```
这样ip netns add xxx命令结束后，/proc/self/ns/net对应的namespace也不会消失。
在每个进程所属的/proc/$pid/ns下面都可以看到每个ns对应的id，相同的ns具有相同的id。
```bash
root@royluo-VirtualBox:/proc/4232/ns# ls -l
total 0
lrwxrwxrwx 1 root root 0 12月 13 02:12 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 12月 13 02:12 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 12月 13 02:12 net -> net:[4026531956]   <<<<<<
lrwxrwxrwx 1 root root 0 12月 13 02:12 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 12月 13 02:12 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 12月 13 02:12 uts -> uts:[4026531838]
root@royluo-VirtualBox:/proc/4232/ns# 
```
通过这样的机制创建了新的netns。

### ip netns exec xxx command
在指定的ns中执行command
```bash
static int netns_exec(int argc, char **argv)
{
	...
	//打开指定ns在/var/run/netns/xxx文件，该文件之前在ip netns add中创建，绑定到一个指定的ns
	snprintf(net_path, sizeof(net_path), "%s/%s", NETNS_RUN_DIR, name);
	netns = open(net_path, O_RDONLY);
	...
	//通过setns，进入指定的ns的namespace中
	setns(netns, CLONE_NEWNET)  <<<<<setns() API
}

内核代码实现：
SYSCALL_DEFINE2(setns, int, fd, int, nstype)
{
	...
	//得到fd对应的/proc文件系统中的信息
	file = proc_ns_fget(fd);
	......
	ei = get_proc_ns(file_inode(file));
	//创建一个新的ns
	new_nsproxy = create_new_namespaces(0, tsk, current_user_ns(), tsk->fs);
	...
	//在新的ns中挂载指定类型的ns
	ops->install(new_nsproxy, ei->ns);
	....
	//切换到新的namespace中
	switch_task_namespaces(tsk, new_nsproxy);
}
```


### ip link add xxx type veth peer name yyy
创建veth接口，veth是内核中比较特殊的接口，其以pair形式存在，即该设备的接口成对出现，如图中的Eeth0-B<------->VETH-B, Eeth0-S<------->VETH-S.其实现代码在linux源码中以驱动类型出现：driver/net/veth.c
```bash
static int veth_newlink(struct net *src_net, struct net_device *dev,
			 struct nlattr *tb[], struct nlattr *data[])
{
	...
	priv = netdev_priv(dev);       <<<<<<<
	rcu_assign_pointer(priv->peer, peer);  
                                     
	priv = netdev_priv(peer);      <<<<<<<成对的创建和注册
	rcu_assign_pointer(priv->peer, dev);   
	...
}
```
接口上的数据流向的特殊性主要体现在veth_xmit()
```bash
static netdev_tx_t veth_xmit(struct sk_buff *skb, struct net_device *dev)
{
	struct veth_priv *priv = netdev_priv(dev);
	struct net_device *rcv;
	int length = skb->len;

	rcu_read_lock();
	rcv = rcu_dereference(priv->peer);  <<<<<取出peer dev

	dev_forward_skb(rcv, skb)  <<<<<peer dev直接收取dev发送的数据
}
```
veth就像一个管道，两端接口可以分配给不同的namespace，实现两个不同ns之间的通信


### ip link set xxx netns yyy
将接口xxx配置给netns yyy，如何取得yyy的namespace呢？主要通过打开/var/run/netns/yyy文件，得到fd后，通过内核对/proc文件操作得到ns id，最后将其设置给net_dev，使指定的netdev属于指定的ns。
```bash
int get_netns_fd(const char *name)
{
	char pathbuf[MAXPATHLEN];
	const char *path, *ptr;

	path = name;
	ptr = strchr(name, '/');
	if (!ptr) {
		snprintf(pathbuf, sizeof(pathbuf), "%s/%s",
			NETNS_RUN_DIR, name );
		path = pathbuf;
	}
	return open(path, O_RDONLY);
}
```

### network namespace内核的支持与实现
未完待续......