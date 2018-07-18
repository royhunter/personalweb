title: OVS02-VXLAN内核相关实现
date: 2018-07-17 20:15:34
tags: Network
categories: Virtualization
---
VXLAN内核相关实现分析.

<!--more-->


## OVS VXLAN tunnel 相关配置命令
```
$ ovs-vsctl add-br br0
$ ovs-vsctl add-port br0 vxlan1 -- set interface vxlan1 type=vxlan \
    options:remote_ip=192.168.1.2 options:key=flow options:dst_port=8472
#ovs-vsctl show
b62cd47f-50be-4d3e-a0a9-5ad3169added
Bridge "br0"
    Port "vxlan1"
        Interface "vxlan1"
            type: vxlan
            options: {csum="true", key=flow, remote_ip="192.168.1.2", tos=inherit}

```

```
options: key=100    <<<<vni为100
options: key=flow   <<<<vni由流表来处理
options: remote_ip=192.168.1.2   <<<< remote vxlan vtep
options: dst_port=8472  <<<< vxlan vtep UDP监听端口
```

## 内核结构关系
![](http://7j1zal.com1.z0.glb.clouddn.com/vxlan.png)

## 几个关键的数据结构
#### OVS支持的vport的类型，本文主要分析VXLAN
```
enum ovs_vport_type {
	OVS_VPORT_TYPE_UNSPEC,
	OVS_VPORT_TYPE_NETDEV,   /* network device */
	OVS_VPORT_TYPE_INTERNAL, /* network device implemented by datapath */
	OVS_VPORT_TYPE_GRE,      /* GRE tunnel. */
	OVS_VPORT_TYPE_VXLAN,	 /* VXLAN tunnel. */     <<<<<<<VXLAN port
	OVS_VPORT_TYPE_GENEVE,	 /* Geneve tunnel. */
	OVS_VPORT_TYPE_LISP = 105,  /* LISP tunnel */
	OVS_VPORT_TYPE_STT = 106, /* STT tunnel */
	__OVS_VPORT_TYPE_MAX
};
```

#### vport  ovs-vsctl add-port创建，ovs port在kernel对应一个vport
```
struct vport {
	struct net_device *dev;
	struct datapath	*dp;   <<<<<<<<datapath, 也就是bridge
	struct vport_portids __rcu *upcall_portids;
	u16 port_no;

	struct hlist_node hash_node;
	struct hlist_node dp_hash_node;
	const struct vport_ops *ops;

	struct list_head detach_list;
	struct rcu_head rcu;
};
```

#### vxlan dev, 一个vxlan vport对应一个vxlan dev
```
/* Pseudo network device */
struct vxlan_dev {
	struct hlist_node hlist;	/* vni hash table */
	struct list_head  next;		/* vxlan's per namespace list */
	struct vxlan_sock __rcu *vn4_sock;	/* listening socket for IPv4 */
#if IS_ENABLED(CONFIG_IPV6)
	struct vxlan_sock __rcu *vn6_sock;	/* listening socket for IPv6 */
#endif
	struct net_device *dev;
	struct net	  *net;		/* netns for packet i/o */
	struct vxlan_rdst default_dst;	/* default destination */
	u32		  flags;	/* VXLAN_F_* in vxlan.h */

	struct timer_list age_timer;
	spinlock_t	  hash_lock;
	unsigned int	  addrcnt;

	struct vxlan_config	cfg;

	struct hlist_head fdb_head[FDB_HASH_SIZE];
};
```
#### VTEP, UDP socket 信息
```
/* per UDP socket information */
struct vxlan_sock {
	struct hlist_node hlist;
	struct socket	 *sock;
	struct hlist_head vni_list[VNI_HASH_SIZE];  <<<<<Virtual Network hash table,用于索引vxlan_dev
	atomic_t	  refcnt;
	u32		  flags;
#ifdef HAVE_UDP_OFFLOAD
	struct udp_offload udp_offloads;
#endif
};

```

## 主要的流程
### 创建
```
ovs_vxlan_tnl_init
    ovs_vport_ops_register(&ovs_vxlan_netdev_vport_ops)
        .create			= vxlan_create,
        .send			= vxlan_xmit,
    
    ovs_vport_add()           //vport.c
        ops = ovs_vport_lookup()
        ops->create()
    
    vport_ops.create: vxlan_create
        vxlan_tnl_create();
            vport = ovs_vport_alloc()
            dev = vxlan_dev_create
                rpl_vxlan_dev_create(vxlan_link_ops)     // vxlan.c
                    rtnl_create_link()
                    vxlan_dev_configure()
                        vxlan_ether_setup()
                            dev->netdev_ops = &vxlan_netdev_ether_ops;
                        register_netdevice()    // Linux net device
                    rtnl_configure_link()
                    
        ovs_netdev_link();
            vport->dev = dev;
```

``` 
dev->netdev_ops = &vxlan_netdev_ether_ops;   // vxlan.c
.ndo_open		= vxlan_open,
.ndo_start_xmit		= vxlan_dev_xmit,
```

```
vxlan_open()  //vxlan.c
    vxlan_sock_add(vxlan_dev)  //vxlan_dev 与 udp sock的关联
        __vxlan_sock_add()
            vxlan_find_sock() // 基于network namespace, address family and UDP port, shareable flag查找
            vxlan_socket_create() // if find_socket is null
                sock = vxlan_create_sock()
                    udp_sock_create()
                        rpl_udp_sock_create4() //udp_tunnel.c
                    tunnel_cfg.encap_rcv = vxlan_rcv;    
                setup_udp_tunnel_sock()
                    rpl_setup_udp_tunnel_sock() //Setup the given (UDP) sock to receive UDP encapsulated packets    udp_tunnel.c
        vxlan_vs_add_dev(vxlan_sock, vxlan_dev)            
```



### 接收
```
vxlan_rcv() //Callback from net/ipv4/udp.c to receive packets  vxlan.c
    struct vxlan_dev * vxlan = vxlan_vs_find_vni()  //这里好像只会找到第一个匹配的vxlan_dev
    netdev_port_receive(skb) //skb的dev为vxlan_dev    vxlan解封装
        vport = ovs_netdev_get_vport(skb->dev);
        ovs_vport_receive(vport)   // vport.c
            ovs_flow_key_extract()
            ovs_dp_process_packet()  <<<<<<<<<进入内核流表核心查询流程
    
```

### 发送
```
ovs_vport_send()
    vport->ops->send(skb);
    vxlan_xmit()
        rpl_vxlan_xmit()
            vxlan_xmit_one()
                udp_tunnel_xmit_skb()    // udp_tunnel.c
                    rpl_iptunnel_xmit()  // ip_tunnels_core.c
                        ip_local_out()
```
