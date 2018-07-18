title: OVS01-OVS基础-安装、配置、Simple Case
date: 2017-04-18 21:26:32
tags: Network
categories: Virtualization
---
OpenVSwitch基础-安装、配置、Simple Topology Test Case.
<!--more-->

## OVS Build and Install
1. Download source code
```
$wget http://openvswitch.org/releases/openvswitch-2.7.0.tar.gz
```


2. Build
```
$tar zxvf openvswitch-2.7.0.tar.gz
$cd openvswitch-2.7.0
$./boot.sh
$./configure --with-linux=/lib/modules/`uname -r`/build 2>/dev/null
$make && make install
```

3. Insmod openvswitch.ko
```bash
$modprobe datapath/linux/openvswitch
root@dev:~# lsmod |grep open
openvswitch            70005  0 
vxlan                  30555  1 openvswitch
gre                     4796  1 openvswitch
libcrc32c               1212  2 dm_persistent_data,openvswitch

```



4. Start

Create ovsdb 
```bash
$mkdir -p /usr/local/etc/openvswitch
$ovsdb-tool create /usr/local/etc/openvswitch/conf.db vswitchd/vswitch.ovsschema
```

Launch ovsdb-server
```bash
$ovsdb-server -v --remote=punix:/usr/local/var/run/openvswitch/db.sock --remote=db:Open_vSwitch,Open_vSwitch,manager_options --private-key=db:Open_vSwitch,SSL,private_key --certificate=db:Open_vSwitch,SSL,certificate --bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --pidfile --detach --log-file
```

Init ovsdb
```bash
$ovs-vsctl --no-wait init
```

Start ovs-vswitchd
```bash
$ovs-vswitchd --pidfile --detach --log-file
```

## OVS Commands
### ovs-vsctl
Open vSwitch commands:
```
init   //initialize database, if not yet initialized
show   //print overview of database contents
```

Bridge commands:
```
add-br BRIDGE               //create a new bridge named BRIDGE
del-br BRIDGE               //delete BRIDGE and all of its ports
list-br                     //print the names of all the bridges
```

Port commands:
```
list-ports BRIDGE           //print the names of all the ports on BRIDGE
add-port BRIDGE PORT        //add network device PORT to BRIDGE
del-port [BRIDGE] PORT      //delete PORT (which may be bonded) from BRIDGE
```

```
ovsdb-tool show-log
```

## Simple Topology
![](http://7xoxkz.com1.z0.glb.clouddn.com/ns_3.PNG)


## Configure for using OVS

```bash
ovs-vsctl add-br br0
ovs-vsctl show
ip link add eth0-b type veth peer name veth-b
ip link add eth0-s type veth peer name veth-s
ovs-vsctl add-port br0 veth-b
ovs-vsctl add-port br0 veth-s
ovs-vsctl show
```

```bash
ip netns add beijing
ip netns add shanghai
ip netns
ls /var/run/netns
```

```bash
ip link set veth-b up
ip link set veth-s up
ip link set br0 up
```

```bash
ip link set eth0-b netns beijing
ip link set eth0-s netns shanghai
ip netns exec beijing ip link
ip netns exec shanghai ip link
ip netns exec beijing ip link set dev lo up
ip netns exec beijing ip link set dev eth0-b up
ip netns exec beijing ip address add 10.0.0.1/24 dev eth0-b
ip netns exec shanghai ip link set dev lo up
ip netns exec shanghai ip link set dev eth0-s up
ip netns exec shanghai ip address add 10.0.0.2/24 dev eth0-s
```

```bash
ip netns exec beijing ping 10.0.0.2
ip netns exec shanghai ping 10.0.0.1
```

```bash

root@dev:~/ovs# ovs-vsctl list-br
br0

root@dev:~/ovs# ovs-vsctl list-ports br0
veth-b
veth-s

root@dev:~/ovs# ovs-vsctl show
2445a7ca-a2c4-4767-a627-15804082244c
    Bridge "br0"
        Port "br0"
            Interface "br0"
                type: internal
        Port veth-s
            Interface veth-s
        Port veth-b
            Interface veth-b

root@dev:~# ovs-ofctl dump-ports br0
OFPST_PORT reply (xid=0x2): 3 ports
  port LOCAL: rx pkts=12, bytes=1024, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=2988, bytes=316309, drop=0, errs=0, coll=0
  port  2: rx pkts=77, bytes=7242, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=85, bytes=7890, drop=0, errs=0, coll=0
  port  3: rx pkts=77, bytes=7242, drop=0, errs=0, frame=0, over=0, crc=0
           tx pkts=84, bytes=7800, drop=0, errs=0, coll=0
           
root@dev:~# ovs-ofctl show br0
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000facc8971cc43
n_tables:254, n_buffers:0
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 2(veth-b): addr:82:1a:0f:d0:22:2a
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 3(veth-s): addr:92:27:da:1d:ca:65
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(br0): addr:fa:cc:89:71:cc:43
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
root@dev:~#

root@dev:~# ovs-ofctl dump-flows br0
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=4064.643s, table=0, n_packets=3385, n_bytes=354447, idle_age=0, priority=0 actions=NORMAL
```



```bash
ovs-vsctl show
ovs-appctl fdb/show br0
ovs-ofctl show br0
ovs-ofctl dump-flows br0
ovs-ofctl dump-flows br0 table=0
ovs-ofctl dump-flows br0 table=1
ovs-vsctl list Bridge
ovs-vsctl list Port
ovs-vsctl list Interface
```
