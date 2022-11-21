高压缩比和无损失的压缩会需要进行大量的计算，消耗CPU。

之前的研究主要靠GPU或FPGA来进行硬件加速。

本论文研究了一种ASIC-accelerated compression。（ASIC：Application Specific Integrated Circuit, compression accelerators, such as Intel R QuickAssist Technology (QAT)）

将因特尔公司的QAT加速卡和文件系统ZFS结合起来，在文件系统中实现了快速的gzip压缩。

QZFS中有一个**压缩引擎**，能够根据文件的size和可压缩性智能地选择压缩算法。比如QAT加速的gzip以及软件实现的一些其他压缩算法。或者直接不采取压缩(OFF).

QZFS中有一个**QAT offloading模块**，这个模块可以利用向量IO模型重构文件系统里的文件的数据块，使得QAT卡对这种重构后的文件进行压缩时不会产生额外的内存复制耗时操作。

和软件版的压缩比起来，QZFS在一些压力测试软件中表现得加速了5、6倍。

数据压缩在应用层、文件系统层、data block层都可以进行。但是在文件系统层以及更下面层进行数据压缩，则必须是无损的算法（否则会对上层应用造成不可知的后果），然而这些算法又会消耗大量计算，目前暂时没有在底层进行数据压缩的硬件加速的实际solution。QZFS就是一次在文件系统层次的压缩硬件加速的尝试（**PS**：这篇论文貌似就是做了一个很大的实验，把QAT用到了ZFS中而已）。

ZFS是高级文件系统，是文件系统和卷管理器的结合，提供了数据完整性、RAIZ-Z、Copy on write、数据加密、数据压缩等特性。

QZFS使用的是gzip算法（PS：gzip、bzip2、xz）

**QZFS就是把ZFS中调用压缩函数的部分改成了两个模块**，1.压缩引擎，2.offloading模块。

TCO：Total Cost of Ownership，意思就是公司的总开销。

论文中使用SAMtools这一工具对150GB的基因序列数据进行排序操作，以此来测试性能。作者通过这个测试，发现Lz4和gzip相比，gzip的CPU耗时更大、压缩比更大，因此把gzip的计算搬到QAT上后效果更明显，因此作者选择了gzip算法。

把数据转移到QAT中进行计算是有开销的，因此进行压缩前需要评估一个文件是否值得被送到QAT中进行压缩，或者是否该被压缩。

QZFS既可以作为local文件系统，也可以作为分布式文件系统Lustre的back-end文件系统。

由于内存中的数据在物理上可能不是连续的，因此不能直接通过DMA发送给QAT。作者采用了一种vectored IO Model来将多个buffer中的数据整合成a single data stream再送到QAT中进行压缩，或者从QAT中读出解压后的a single data stream再将其organized to multiple flat buffers.

实验环境：CentOS with Linux Kernel 3.10.

作者的QAT加速已经被整合到ZFS官方的0.7.0版本中，这篇论文是之后才发的。



**压缩引擎**又分为**Compressibility Checker**和**Algorithm Checker**。

ZFS的压缩比至少为 (100%-12.5%), QZFS的压缩比至少为 (100%-10%)。具体是使用了个res buffer，res buffer的长度为原数据长度的90%，当compressed data overflow这个buffer，则压缩会停止，ZIO压缩模块会直接返回原数据作为结果。

作者说QZFS会记录每个文件的原数据长度。

压缩引擎中的record size记录了一个block的最大值，即压缩一个文件是分成多个block进行压缩的，而这个block最大值就是record size。

> 文章中说ZFS的record size默认为128KB，但是下一段立马说QZFS允许最大的source data size为1MB。一个是record size（一次压缩的大小），一个是FIO block size（一次IO的大小），具体区别如何论文中没说。

对于小于4KB的文件，QZFS采用软件的方法进行压缩。

Algorithm Checker就是用了个scalable vector把所有算法整理在一起方便调用。



**Offloading模块**采用SGL结构体来组织多段分散的物理内存，QAT accelerator通过解析这个SGL结构体就可以通过DMA操作逐段获取数据。

SGL结构体中每个flat buffer实际就是一个page（物理页），但是注意其中的物理数据并不一定是按页对齐的，例如a 11KB source data may correspond to four physical pages (2KB+4KB+4KB+1KB)

```
numBuffers = ($S_{src}$ >> PAGE_SHIFT) + 2
```

构建SGL结构体的详细过程：假设$P_{src}$是当前虚拟地址指针，占用的全是内核内存（非用户内存空间）。判断该页是vmalloc区域还是kmalloc区域，如果是vmalloc则调用vmalloc_to_page函数获取page结构体，否则如果是kmalloc则调用virt_to_page获取page结构体。获取到page结构体后就等于获得了实际的物理地址，然后建立起flat buffer再组织到SGL中即可，最后把$P_{src}$换到下一个位置。

这里作者在建立flat buffer的时候又调用了kmap（原文说是建立永久内存映射，但网上查了一下显示kmap只是帮助page找到vaddr而已，这里不是已经有vaddr $P_{src}$了吗）

QAT卡需要设置一片连续的缓冲区域来进行压缩or解压操作，文中设置的大小为2MB，即2倍的原数据最大值，因为在压缩过程中有可能临时数据大小会超过原数据大小。

QZFS当需要进行很多压缩or解压操作时，还是会把部分任务交给cpu做。

Offloading模块在完成了**数据组成SGL**、**建立QAT卡缓冲区**两个任务，会进行和QAT卡交互的任务，具体是用**QAT compression session结构体**来存储一些交互的参数和数据。

如果QAT加速过程中出了错误，则QZFS会清除并重建之前介绍的结构体，并禁用QAT加速功能直到QAT重启完毕。

压缩后的数据都有个头header，header会表明这是用的哪种压缩算法做的压缩。

**数据压缩**三步骤：

1. 在output SGL结构体中生成gzip style header；
2. 调用函数，使得QAT卡接受input SGL和output SGL，将压缩的数据存储到output SGL上；
3. 在output SGL结构体中生成gzip style footer；