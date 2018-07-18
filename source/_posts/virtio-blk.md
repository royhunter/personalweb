title: Virtio-Blk浅析
date: 2014-08-29 21:08:21
tags: virtio
categories: KVM
---
和virtio-network一样，virtio-blk驱动使用Virtio机制为Guest提供了一个高性能的块设备I/O的方法。我们这里看下virtio-blk的实现。

## Linux中的块设备
在介绍virtio-blk之前，先科普下Linux内核中的块设备整体架构。

<!--more-->

### 基本概念
Linux操作系统有三类主要的设备文件：

1. 字符设备：以字节为单位进行顺序I/O操作的设备；
2. 块设备：以块单位接收输入返回，对于I/O请求有对应的缓冲区，可以随机访问，块设备的访问位置必须能够在介质的不同区间前后移动。在块设备中，最小的可寻址单元是扇区，扇区的大小一般是2的整数倍，常见的大小为512个字节；
3. 网络设备：提供网络数据通信服务。

这里主题讨论块设备。

1. 扇区(Sectors)：任何块设备硬件对数据处理的基本单位。通常，1个扇区的大小为512byte。
2. 块(Blocks)：由Linux制定对内核或文件系统等数据处理的基本单位。通常，1个块由1个或多个扇区组成。

### 整体架构

![](http://7narr6.com1.z0.glb.clouddn.com/1.PNG)

相关说明：

1. 通用块层(Generic Block Layer)负责维持一个I/O请求在上层文件系统与底层物理磁盘之间的关系。在通用块层中，**通常用一个bio结构体来对应一个I/O请求**。
2. 驱动对块设备的输入或输出(I/O)操作，都会向块设备发出一个请求，在驱动中用**request**结构体描述。但对于一些磁盘设备而言请求的速度很慢，这时候内核就提供一种队列的机制把这些I/O请求添加到队列中（即：请求队列），在驱动中用**request_queue**结构体描述。
3. I/O调度层(I/O Scheduler Layer)的作用:在向块设备提交这些请求前内核会先执行请求的合并和排序预操作，以提高访问的效率，然后再由内核中的I/O调度程序子系统来负责提交  I/O 请求，  调度程序将磁盘资源分配给系统中所有挂起的块I/O请求，其工作是管理块设备的请求队列，决定队列中的请求的排列顺序以及什么时候派发请求到设备。
4. 对于每一个独立的磁盘设备或者分区，Linux提供一个**gendisk**数据结构体，用于对底层物理磁盘进行访问。在gendisk中有一个硬件操作结构指针，为**block_device_operations**结构体。

当多个请求提交给块设备时，执行效率依赖于请求的顺序。如果所有的请求是同一个方向（如：写数据），执行效率是最大的。内核在调用块设备驱动程序例程处理请求之前，先收集I/O请求并将请求排序，然后将连续扇区操作的多个请求进行合并以提高执行效率，对I/O请求排序的算法称为电梯算法（elevator algorithm）。电梯算法在I/O调度层完成。内核提供了不同类型的电梯算法，电梯算法有：

1. noop（实现简单的FIFO，基本的直接合并与排序）；
2. anticipatory（延迟I/O请求，进行临界区的优化排序）；
3. Deadline（针对anticipatory缺点进行改善，降低延迟时间）；
4. Cfq（均匀分配I/O带宽，公平机制）。

### 数据结构

1. 块设备对象结构block_device
内核用结构block_device实例代表一个块设备对象，如：整个硬盘或特定分区。如果该结构代表一个分区，则其成员bd_part指向设备的分区结构。如果该结构代表设备，则其成员bd_disk指向设备的通用硬盘结构gendisk
当用户打开块设备文件时，内核创建结构block_device实例，设备驱动程序还将创建结构gendisk实例，分配请求队列并注册结构block_device结构。

2. 通用硬盘结构 gendisk
结构体gendisk代表了一个通用硬盘（generic hard disk）对象，它存储了一个硬盘的信息，包括请求队列、分区链表和块设备操作函数集等。块设备驱动程序分配结构gendisk实例，装载分区表，分配请求队列并填充结构的其他域。
支持分区的块驱动程序必须包含 头文件，并声明一个结构gendisk，内核还维护该结构实例的一个全局链表gendisk_head，通过函数add_gendisk、del_gendisk和get_gendisk维护该链表。

3. 请求结构request
结构request代表了挂起的I/O请求，每个请求用一个结构request实例描述，存放在请求队列链表中，由电梯算法进行排序，每个请求包含1个或多个结构bio实例。

4. 请求队列结构request_queue
每个块设备都有一个请求队列，每个请求队列单独执行I/O调度，请求队列是由请求结构实例链接成的双向链表，链表以及整个队列的信息用结构request_queue描述，称为请求队列对象结构或请求队列结构。它存放了关于挂起请求的信息以及管理请求队列（如：电梯算法）所需要的信息。结构成员request_fn是来自设备驱动程序的请求处理函数。

5. Bio结构
通常1个bio对应1个I/O请求，IO调度算法可将连续的bio合并成1个请求。所以，1个请求可以包含多个bio。
**bio为通用层的主要数据结构，既描述了磁盘的位置，又描述了内存的位置，是上层内核vfs与下层驱动的连接纽带**。

### 总结

块设备的I/O操作方式与字符设备存在较大的不同，因而引入了
request_queue、request、bio等一系列数据结构。在整个块设备的I/O操作中，贯穿于始终的就是“请求”，字符设备的I/O操作则是直接进行访问，
为提高性能，块设备的I/O操作会进行排队和整合。
驱动的任务是处理请求，对请求的排队和整合由I/O调度算法解决，因此，块设备驱动的核心就是请求处理函数或“制造请求”函数。

## virtio-blk

### 初始化
相关代码位于：drivers/block/virtio_blk.c

```c
static int __init init(void)
{
	int error;

	virtblk_wq = alloc_workqueue("virtio-blk", 0, 0);
	if (!virtblk_wq)
		return -ENOMEM;

	major = register_blkdev(0, "virtblk");
	if (major < 0) {
		error = major;
		goto out_destroy_workqueue;
	}

	error = register_virtio_driver(&virtio_blk);
	if (error)
		goto out_unregister_blkdev;
	return 0;

out_unregister_blkdev:
	unregister_blkdev(major, "virtblk");
out_destroy_workqueue:
	destroy_workqueue(virtblk_wq);
		return error;
}
```

通过register_blkdev()向块设备层注册一个块设备，并同时通过register_virtio_driver向virtio层注册了virtio_blk driver。前面virtio分析中说到virtio设备层是一个PCI设备接口层。所以这里的virtio blk都是建立在pci接口之上的。

当Qemu启动Guest时指定了使用virtio blk设备时，virtio_blk结构中注册的probe函数会在启动过程中调用到来初始化virtio blk设备。具体的virtblk_probe()作用为：

- 分配struct virtio_blk结构，代表一个virtio blk设备

```c
vdev->priv = vblk = kmalloc(sizeof(*vblk), GFP_KERNEL);
```

- 分配virtqueue，这里不同于virtio-net设备，只使用了一个virtqueue

```c
init_vq(vblk);
```

- 分配gendisk结构，代表了virtio blk物理磁盘

```c
vblk->disk = alloc_disk(1 << PART_BITS);
```

- 分配request_queue结构，从属于virtio-blk的gendisk结构下

```c
q = vblk->disk->queue = blk_mq_init_queue(&virtio_mq_reg, vblk);
```

对request的操作处理函数都在virtio_mq_reg结构的virtio_mq_ops中：

```c
static struct blk_mq_ops virtio_mq_ops = {
	.queue_rq	= virtio_queue_rq,
	.map_queue	= blk_mq_map_queue,
	.alloc_hctx	= blk_mq_alloc_single_hw_queue,
	.free_hctx	= blk_mq_free_single_hw_queue,
	.complete	= virtblk_request_done,
};
```

- request的存储区vbr初始化，结构依旧是scatter-list形式

```c
blk_mq_init_commands(q, virtblk_init_vbr, vblk);
```

内核中块I/O操作的基本容器由bio结构体表示，该结构体代表了正在现场的（活动的）以片段（segment）链表形式组织的块I/O操作。一个片段是一小块连续的内存缓冲区。这样的好处就是不需要保证单个缓冲区一定要连续。所以通过片段来描述缓冲区，即使一个缓冲区分散在内存的多个位置上，bio结构体也 能对内核保证I/O操作的执行，这样的就叫做聚散I/O.

- 分配virtio blk的磁盘名称

```c
virtblk_name_format("vd", index, vblk->disk->disk_name, DISK_NAME_LEN);
```

使用virtio_blk驱动的磁盘显示为“/dev/vda”，这不同于IDE硬盘的“/dev/hda”或者SATA硬盘的“/dev/sda”这样的显示标识。

- 完善disk信息，将virtio blk的disk信息注册至块设备层同一管理 

```c
vblk->disk->major = major;
vblk->disk->first_minor = index_to_minor(index);
vblk->disk->private_data = vblk;
vblk->disk->fops = &virtblk_fops;
vblk->disk->driverfs_dev = &vdev->dev;
vblk->index = index;
add_disk(vblk->disk);
```

![](http://7narr6.com1.z0.glb.clouddn.com/2.PNG)

### 数据处理

#### 后端----->前端

virtio_blk结构体中的gendisk结构的request_queue队列接收block层的bio请求，按照request_queue队列默认处理过程，bio请求会在io调度层转化为request，然后进入request_queue队列，最后调用virtblk_request将request转化为vbr结构。

	virtio_queue_rq()                <---- 注册于request_queue结构的queue_rq成员
		--->blk_rq_map_sg()          <---- 将vbr填入scatter-list中
		--->__virtblk_add_req()       
			--->virtqueue_add_sgs()  <---- scatter-list成员入virtqueue的vring
		--->virtqueue_kick           <---- 通知前端

最后由Qemu接管处理。

#### 前端----->后端
Qemu处理过vdr后会将它加入到virtio_ring的request队列，并发一个中断给队列，队列的中断响应函数vring_interrupt调用队列的回调函数virtblk_done；

	virtblk_done()
		--->blk_mq_complete_request()

最后由request_queue注册的complete函数virtblk_request_done()处理，通过blk_mq_end_io()通告块设备层IO结束。

### request生命周期的图示

![](http://7narr6.com1.z0.glb.clouddn.com/3.PNG)

## 参考资料

1. “深入理解Linux内核”
2. "2012-forum-virtio-blk-performance-improvement.pdf"