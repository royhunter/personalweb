title: Virtio-Blk性能加速方案
date: 2014-08-31 18:52:20
tags: virtio
categories: KVM
---
在前面"virtio-blk浅析"一文中我们看到了virtio-blk设备的整体架构，其核心和virtio-net设备一样通过将针对磁盘操作的request命令放入virtqueue的vring中进行传输，由于要通过用户态程序的Qemu周转一次，再到host kernel层，性能有一定的损耗。为提升virtio-blk的性能，出现了两种方案。

1. Bio-based virtio-blk
2. vhost-blk

下面我们来看下这两种加速方案。

<!--more-->

![](http://7narr6.com1.z0.glb.clouddn.com/4.PNG)

## virtio-blk acceleration
在Linux3.15主线版本上还没有出现virtio-blk加速方案的代码。相关的代码在一个git@github.com:asias/linux.git blk.vhost-blk的分支上，由Red Hat的Asias He实现并托管在Github，大家可以通过这个命令下载：

```bash
git clone git@github.com:asias/linux.git
git@github.com:asias/qemu.git blk.vhost-blk
```

一个是集成了加速方案的Linux kernel代码，一个是支持加速方案的Qemu代码。

### Bio-based virtio-blk
代码位于： /drivers/block/virtio-blk.c
从如下层次图中可以看出，Bio-based加速方案通过在通用块设备层注册了make_request_fn()函数，具体实现为virtblk_make_request()，其将bio组成vbr结构，跳过了IO Scheduler层将命令写入了virtqueue的vring中的，进行kick。

![](http://7narr6.com1.z0.glb.clouddn.com/5.PNG)

代码调用流程：

	virtblk_make_request()
		--->vbr = virtblk_alloc_req(vblk, GFP_NOIO);
		--->virtblk_bio_send_data(vbr);
			--->sg_set_buf()
			--->virtblk_add_req()
				--->virtqueue_add_buf()
				--->virtqueue_kick()

而使用Bio-based virtio-blk机制需要通过命令modprobe virtio-blk use_bio = 1.

###　Vhost-blk
类似于vhost-net，vhost-blk提供了virtio-blk在host kernel层的数据面加速。
代码位于： drivers/vhost/

![](http://7narr6.com1.z0.glb.clouddn.com/6.PNG)

和virtio-net一样，vhost通过创建“vhost-pid”的内核线程来实现vring中数据的处理及Gust和Host的双向事件通知。

## 参考资料

1. "2012-forum-virtio-blk-performance-improvement.pdf" --- Asias He

