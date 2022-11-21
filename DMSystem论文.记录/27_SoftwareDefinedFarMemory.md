## Abstract

我们提出了一种新颖的软件定义的远程内存方法，它主动压缩冷内存页面以有效地在软件中创建远程内存层。

2022-05-31：**Far Memory**不是远程内存的意思，而是离CPU较远的内存。

## 1 Introduction

灵活地分配各种computer资源需要各种类型的资源都足够灵活才行，任何一种不能灵活分配的类型的资源都会限制这种模式的实现。

近年来DRAM就成为一种不能灵活scaling的资源，特别是近年来大数据应用的盛行，摩尔定律的终结。

一个有希望的方向就是引入远程内存作为DRAM和底层Flash之间的中间层。

现代服务器设备和上面的应用对memory有以下要求：

1. Near-zero tolerance to application slowdown.

   就是说application不能减速，否则客户不用，而且还有些其它负面影响。**即使是几个百分点的减速也可以抵消所有潜在的TCO节省**。

2. Heterogeneity of applications.

   在服务器上运行的应用程序变得越来越多样，因此不能要求对特定应用的优化和改动，必须是一种透明而强大的机制来有效利用远程内存。

3. Dynamic cold memory behavior.

   就是说服务器上的应用对内存利用率呈时间上的变化，因此，近内存和远内存之间的最佳比率不仅取决于在给定时间运行的工作负载，而且还会随着时间而变化。

作者的贡献：

1. 分析了真实时间服务器的内存使用情况，发现在不同cluster之间的cold memory比例从1%到61%，即使在相同cluster内cold memory比例从1%到52%。

2. 设计了一种software-defined approach to far memory. 

   这种方案差不多就是将zswap的模式用到了远程内存，并且可以主动地将cold pages驱逐到远程内存。

   > Specifically, we demonstrate that zswap [1], a Linux kernel mechanism that stores memory compressed in DRAM, can be used to implement software-defined far memory that provides single-digit μs of latency at tail.

