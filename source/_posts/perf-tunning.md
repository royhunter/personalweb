title: datapath性能优化总结
date: 2018-01-10 21:26:32
tags: Network
categories: Virtualization
---
一些数据面(报文转发)性能提升的points.
<!--more-->

## Tools
1. Perf     linux内核自带了一个性能分析工具
2. Dperf    CPU(Octeon)提供的各种硬件监测指标（通过读取各种寄存器）

监控因子：
1. cache miss 
2. cache hit
3. function run cycle
4. instruction num
5. Branch miss

原理：运行时设置，通过修改函数入口和出口处的跳转至监控代码处。


## 系统级的优化
1. 减少tlb miss，通过btlb实现；
2. 代码段热点流程函数指定在同一个段中，减少代码跨度大导致的指令cache miss；
3. 根据cache的策略调整地址空间；很多cpu用虚拟地址索引，物理地址匹配cache 行，多核情况下，可将同构的多核代码虚拟地址错开；
4. 编译选项-o3;
5. 指令预取；



## 代码级优化
1. 减少锁冲突； 能local的数据尽量local； 采用无锁优化算法；
2. 指令上的优化；比如memset可用64位指令赋值来优化；
3. 关键位置代码使用短小精悍的内嵌汇编代码实现；
4. 用移位实现乘除法运算；
5. inline函数； 宏定义；减少压栈出栈的时间开销；
6. static增加内联；
7. 固定查表优化；
8. likely/unlikely 增加分支预测准确度；
9. 数据结构cache line对齐；
9. 业务算法优化；

