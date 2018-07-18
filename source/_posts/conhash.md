title: 一致性哈希算法(Consistent Hashing)及Golang实现
date: 2016-02-25 23:43:42
tags: Distribute
categories: Algorithm
---
在提供分布式集群服务的系统中，在索引对象的存储节点的方法中比较常用的方法即简单的hash集群，即使用存储对象的hash值来索引指定的存储节点。
但其中存在几个问题：
1. 若其中一个节点crash了，再使用固定hash模数时，大部分的存储对象的存储节点都会发生变化，若作存储对象迁移的话数据量非常大。
2. 物理节点个数少时，由于hash函数的计算特性会导致索引分布在某些情况下极度不均匀。
<!--more--> 

一致性hash能够比较好的解决此类问题，一致性哈希算法（Consistent Hashing）最早在David Karger，Eric Lehman等人的论文《Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web》中被提出，是当前较主流的分布式哈希表协议之一。

网上有非常多的对该算法的详解，此处不再多说，只贴出下图大概表示下一致性hash的大概结构：
![](http://7xoxc5.com1.z0.glb.clouddn.com/conhash_1.PNG)
将存储节点通过hash方式分布到0---2^32-1的连续环上，存储对象通过hash方式落于环中，取顺时针的第一个存储节点为存储服务节点。为解决均匀性问题，使用一个节点映射多个虚拟节点来使节点均匀在环上均匀分布。

存储对象与存储虚拟节点及真实节点的映射关系：
![](http://7xoxc5.com1.z0.glb.clouddn.com/conhash_2.PNG)

在memcache的client端的libmemcached中已经实现了该算法。以下是自己所作的Golang实现。完整代码可见：[https://github.com/royhunter/ConsistentHash](https://github.com/royhunter/ConsistentHash)

```go
package main


import (
    "fmt"
    "hash/crc32"
    "strconv"
    "sort"
)

type HashFunc func(date []byte) uint32
type HashRing []uint32

func(r HashRing) Len() int { return len(r) }
func(r HashRing) Less(i, j int) bool { return r[i] < r[j] }
func(r HashRing) Swap(i, j int) { r[i], r[j] = r[j], r[i] }

type ConHash struct {
    hashfunc  HashFunc
    hashring  HashRing
    vnodes    int
    node_map_num      map[string] int  //physical node---> node num
    vnode_map_node    map[uint32] string  //vnode key ---->physical node
    
}


func ConHashNew() *ConHash {
    return &ConHash {
        hashfunc: crc32.ChecksumIEEE,
        hashring: HashRing{}, 
        vnodes: 0,
        node_map_num: make(map[string] int),
        vnode_map_node: make(map[uint32] string),
    }
}

func (ch *ConHash) NodeHash(data []byte) uint32 {
    return ch.hashfunc(data) 
}

//vnode format:  "n_name_%d"
func (ch *ConHash) NodeAdd(n_name string, vn_num int) {
    var name string
    var key uint32
    if _,ok := ch.node_map_num[n_name]; ok {
        fmt.Println("node", n_name, "already exist")
        return
    }
    ch.node_map_num[n_name] = vn_num
    for i := 0; i < vn_num; i++ {
        name = n_name + "_" + strconv.Itoa(i)
        key = ch.NodeHash([]byte(name))
        ch.vnode_map_node[key] = n_name
        ch.hashring = append(ch.hashring, key)
        ch.vnodes++
    }
    sort.Sort(ch.hashring)
}

func (ch *ConHash) NodeRemove(n_name string) {
    var name string
    var key uint32

    if _,ok := ch.node_map_num[n_name]; !ok {
       fmt.Println("node", n_name, "not exist") 
    }

    vnode_num := ch.node_map_num[n_name]
    for i := 0; i < vnode_num; i++ {
        name = n_name + "_" + strconv.Itoa(i)
        key = ch.NodeHash([]byte(name))
        for i,v := range ch.hashring {
            if v == key {
                ch.hashring = append(ch.hashring[:i], ch.hashring[i+1:]...)
            }
        }
        delete(ch.vnode_map_node, key)
        ch.vnodes--
    }
    delete(ch.node_map_num, n_name)
}


func (ch *ConHash) NodeLookup(object string) string {
    var key uint32
    var hitindex int

    key = ch.NodeHash([]byte(object))
    index := sort.Search(len(ch.hashring), func(i int) bool { return ch.hashring[i] >= key })
    if index == len(ch.hashring) {
        hitindex = 0
    } else {
        hitindex = index
    }
    hitkey := ch.hashring[hitindex]
    node, _ := ch.vnode_map_node[hitkey]
    return node
}

func (ch *ConHash) NodeGetVnodes() int {
    return ch.vnodes
}
```

C语言实现版本：[http://www.codeproject.com/Articles/56138/Consistent-hashing](http://www.codeproject.com/Articles/56138/Consistent-hashing)

Reference:
1. [http://en.wikipedia.org/wiki/Consistent_hashing](http://en.wikipedia.org/wiki/Consistent_hashing)
2. [https://sourceforge.net/projects/libconhash ](https://sourceforge.net/projects/libconhash)
