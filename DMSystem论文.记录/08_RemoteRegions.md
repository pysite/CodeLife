## Abstract

设计了一组取代RDMA的接口，使得应用可以像使用文件系统的接口就可以访问远端内存，而使用RDMA就十分的复杂繁琐。（使用文件系统的接口只需9行代码就能完成的事，RDMA需要300行代码）



## Introduction

> Remote memory is available now, using RDMA technology over Infiniband or Ethernet.

Remote Memory的两个问题：

1. 没有统一接口，现存技术使用RDMA verbs，但是一些硬件例如Gen-Z和OpenCAPI使用自己的接口来控制访问、映射等操作。并且即使RDMA仍在不断发展中，但是其一些新特性并不是在所有环境下都提供。

2. 难以使用。现存RDMA技术每访问一次远端都需要大量复杂的代码。

   > Code is required to initialize contexts, register memory, establish RDMA connections, create queue-pairs, associate them with connections, transition the queues through various states, exchange RDMA keys, post commands on queues, and poll the queues for completions.

3. RDMA缺少应用所需的naming和location服务。



本论文提供的文件系统叫REGIONFS(regions)，一个进程可以让出其部分内存以文件的形式到REGIONFS中，稍后这部分内存就可以被远端Client进程借用(成为其remote memory)。

除了只使用文件系统接口外，regions还提供了传统文件系统有的功能：name space, timestamp，access control。

作者在设计regions时解决了以下问题：

> To find this balance, we address questions of file semantics, memory allocation, data sharing, memory mapping, page fault preemption, security, data-metadata separation, caching, cache coherence, and sharing granularity, while addressing RDMA limits on memory registration, connections, and keys.

作者的regions就是基于RDMA封装的：

> We have built regions using RDMA in the Linux kernel v4.8



## Related Work

### Same interface, different goal

> 仍是文件系统接口，但目的不是Remote Memory

目的是Remote Storage(不是Memory)，使用RDMA去改善性能，例如Octopus, Crail, Ceph, GlusterFS。

目的是"a file system protocol for RDMA" DAFS.

目的是"run NFS over RDMA" NFS with RDMA.

目的是"manage large local memories" Towards O(1) memory.

### Different interface, same goal

> 目的仍是Remote Memory，但不是文件系统接口

"a kernel interface that offers more flexible protection" LITE.

"使用RDMA实现lock-free read操作" FaRM.

### Different interface, different goal

> Many systems provide remote storage with an interface other than files.

例如KV系统，以及一些常见的数据库都属于remote storage。

### Remote memory applications

CacheDM使用remote memory作为NFS的cache。

Infiniswap使用remote memory作为本地swap的cache。

> 以上这些Application如果使用regions的话代码会简化很多。

### New hardware

> 就是指Disaggregated Memory的出现

Regions可以给内存解耦系统提供简便的接口，但是一些接口的底层实现需要针对具体应用进行修改。



## Assumptions, goals, and motivation

### 假设

1. 假设机器都是通过低延迟、高带宽、可靠性的网络互相连接的。
2.  We assume a single administrative, trust, and fault domain.
3. 假设机器数量为几十到一百台，几千台的那种大型企业不是target environment。

4. 网络分裂(network partition)时有可能发生的，当其发生时系统会暂停。

### 目标

目标是提供一套简单的接口供应用访问远程进程的内存。

### 动机

当前用的是one-sided read and write operations using the verbs library.其问题：

1. verbs operations太过复杂。
2. 除了RDMA其它的library就需依靠具体硬件，例如Omni-Path和Gen-Z。
3. RDMA对网络适配器的资源有限制，比如对连接的cache大小设限。

本文设计的接口就是希望hide这些缺点，来使得上层App无需过多担心下层细节。



## Why files?

> 为什么要设计成文件系统的接口?

1. Well-known
2. 文件系统接口被很多应用使用，例如find, grep, cat, awk等；
3. 所有主流语言都支持文件操作，这些语言要支持regions的话只需要再移植必要的同步库就可。
4. 有很多tool支持插入到文件系统调用之中进行debug, trace等；
5. 文件系统支持目录树，方便管理data；
6. 文件系统支持access control；



## The regions abstraction

