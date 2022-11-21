# LITE Kernel RDMA

## 1 Introduction

native RDMA不适合more general-purposed, heterogeneous, and large-scale datacenter environments的三大理由：

1. datacenter application要想发挥RDMA全部性能，需要定制底层RDMA代码，对代码进行大量改动；
2. 没有对RDMA资源进行管控，比如缺少QoS的管理；
3. RNIC自带的SRAM是有限的（RNIC store protection keys and cache page table entries for user memory regions in its SRAM），不会随着应用的内存使用增大而scale；

LITE就是工作在内核态，为上层应用提供RPC、sync等API的RDMA虚拟层，可以让应用很方便使用RDMA，并且提供performance isolation。

之前的工作主要是将一些work卸载到硬件上，而LITE则是将一些work还是交给OS进行，因为作者发现通过良好的设计，使用内核态中的虚拟层RDMA可以保留native RDMA的性能同时提供很大的便利。

> 评估表明，与原生RDMA和针对某些应用程序定制的现有解决方案相比，LITE提供了相似的延迟和吞吐量，同时提高了灵活性、性能可扩展性、CPU利用率、资源共享和服务质量。

## 2 Background And Issues

### 2.3 Issue 1: Mismatch in Abstractions

RDMA API难用，RDMA性能难发挥出来。

### 2.4 Issue 2: Unscalable Performance

RDMA QP或RDMA MR数量过多就会给RDMA性能带来下降，因为RDMA SRAM是有限的；而使用较大的MR又会造成一定的空间浪费。

当MR较大时（超过4MB），PTE就会装不下RDMA SRAM，当PTE miss时RNIC就必须去Host OS中查询，FaRM有用2GB的huge page，但这样会造成memory fragmentation等一系列副作用。

当QP较多时同样会degrade performance，之前的FaSST采用UD来解决这一问题，但UD是不可靠的且不支持one-sided RDMA。

> 综上，作者认为不该把all priv-iledged functionalities and metadata offload to hardware，需要对RDMA software and hardware stacks进行重构。

### 2.5 Issue 3: Lack of Resource Sharing, Isolation, and Protection

native RDMA缺少资源共享的机制，会加重2.4中提到的问题，比如每对进程之间都得建立QP、CQ等，还必须至少拿一个thread出来不断poll，非常浪费资源。

native RDMA缺少资源管控的机制，不能保证QoS，比如一个进程可以注册很多块MR，使得RNIC性能下降，影响使用同一个RNIC的其它应用。

native RDMA缺少安全机制，比如MR访问是靠lkey和rkey，但rkey一般是靠应用自身传递给远端，这样一般是明文传递，如果rkey泄露出去就可能遭受攻击。并且MR的使用也很不方便，要想修改MR的使用权限必须先de-registered，再重新registered。

## 3 Design Overview

### 3.1 Kernel-Level Indirection

之前的DRAM和DISK在后来都是被virtual memory和file system解决了，因此作者认为RDMA设备也应该被类似的indirection layer解决。而在内核态实现这么个indirection layer则可以管理所有privileged resource，并且在内核态实现如此indirection layer可以既服务内核态也可以服务用户态。

### 3.2 Challenges

1. 保证RDMA性能
2. 保证LITE的通用性
3. 不修改hardware、OS，只做成个loadable module

### 3.3 LITE Overall Architecture

作者在3.11版本内核中用15k行代码实现了LITE作为一个loadable module，系统中的应用可以选择使用LITE也可以选择使用原生RDMA。

LITE支持memory-like operations, RPC and messaging, and synchronization primitives（内核态和用户态都可）。

LITE的RPC和messaging功能基于two-sided RDMA实现。

## 5 LITE RPC

每个RPC function都有自己的ID。

LITE中用LMR（LITE memory region）表示一段内存空间，一个LMR并不是对应一个RDMA MR，一个LMR甚至可以分散在不同的node中。

### 5.1 LITE RPC Mechanism

LITE用两次write-imm操作实现RPC，一次write-imm用于发起RPC请求，一次write-imm用于发送RPC结果，imm会在remote中产生一个ce，remote有一个shared thread一直在busy loop pool imm的cq来负责处理所有的RPC请求，所有client共用一个poll thread。

LITE用32bit imm来传递<RPC function id, start offset>，其中RPC function id标明调用的是哪个函数，start offset表示write将函数参数写到哪个地址。